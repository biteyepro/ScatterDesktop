<template>
    <section>
        <back-bar v-on:back="back" />

        <section class="transfer">

        

        <!-- ADDRESS BOOK -->
        <section class="address-book scroller" v-if="!isSimple">
            <figure class="bg"></figure>

            <section class="head">Select a Key pair</section>

            <section class="search">
                <!-- INPUT -->
                <section class="input">
                    <figure class="icon"><i class="fa fa-search"></i></figure>
                    <input placeholder="Search..." v-model="searchTerms" />
                </section>
            </section>

            <section class="accounts scroller">
                <section class="account" v-for="keypair in vaultEntries" @click="selectKeypair(keypair)">
                    <section class="info">
                        <figure class="name">{{keypair.name}}</figure>
                        <figure class="description">{{keypair.accounts().length}} linked accounts</figure>
                    </section>    
                </section>
            </section>
        </section>



        <!-- TRANSFER DETAILS -->
        <section class="details">
            <section class="actions">
                <figure class="action" @click="switchSimple">
                    {{isSimple ? 'Customize' : 'Simple'}}
                </figure>
                <figure class="action" @click="sending ? null : send()">
                    <span v-if="!sending">Create</span>
                    <span v-else>
                        <i>Please waiting...</i>
                    </span>
                    <!-- <section class="action">
                        <btn :loading="sending"
                            :disabled="!canSend"
                            blue="1"
                            :text="Create"
                            v-on:clicked="send" />
                    </section> -->
                </figure>
            </section>

            <section class="data">

                <section class="inline-inputs">
                    <section class="inputs half">
                        <label>EOSY Account name</label>
                        <input :class="{'with-icon':eosyName}" v-model="eosyName" placeholder="Enter an Account Name" />
                        <transition name="slide-down">
                            <figure class="prefix icon"  v-if="isValidAccount" @click="recipient = ''"><i class="fa fa-check"></i></figure>
                            <figure class="prefix icon" v-if="eosyName && !isValidAccount" @click="recipient = ''"><i class="fa fa-times"></i></figure>
                        </transition>
                        {{invalidAccountCode}}
                    </section>

                    <section class="inputs third">
                        <label>Check code</label> 
                        <input v-model="regCatchaCode" :class="{'with-icon':regCatchaCode}" placeholder="Enter the check code" />
                        <transition name="slide-down">
                            <figure class="prefix icon"  v-if="isValidCaptcha" @click="recipient = ''"><i class="fa fa-check"></i></figure>
                            <figure class="prefix icon" v-if="regCatchaCode && !isValidCaptcha" @click="recipient = ''"><i class="fa fa-times"></i></figure>
                        </transition>
                        <div v-html="captcha" @click="refreshCatcha"></div>
                        {{invalidCatchaCode}}
                    </section>
                </section>

                <div style="height:30px; clear:both;"></div>

                <transition name="slide-left" mode="out-in">
                    <section key="complex" v-if="!isSimple">
                        <div class="breaker"></div>
                        <section class="inline-inputs">
                            <section class="inputs half">
                                <label>Public Key</label>
                                <input v-model="regPubKey" placeholder="Please selected key pair from list" disabled/>
                            </section>                       

                        </section>


                        <transition name="slide-left">
                            <section v-if="!isSimple">
                                <div class="breaker"></div>
                                <section key="customize">
                                    <b>EOSY account customize creation</b>
                                    <p style="margin-top:5px;">
                                        An EOSY account include an account name and a key pair to control the account.
                                        Please choose a key pair from list.                            
                                    </p>
                                </section>
                            </section>
                        </transition>
                    </section>

                    <section key="simple" v-else>
                        <b>EOSY account simple creation</b>
                        <p style="margin-top:5px;">
                            An EOSY account include an account name and a key pair to control the account.
                            The key pair will be generated and saved automatically.                            
                        </p>
                    </section>
                    <!-- STATUS -->
                    <section key="status" v-if="status" class="status">
                        <figure class="title">Status</figure>
                        <figure class="description">{{status}}</figure>
                    </section>

                </transition>

            </section>

        </section>

        </section>
    </section>
</template>

<script>
    import { mapActions, mapGetters, mapState } from 'vuex'
    import * as Actions from '../store/constants';

    import ResourceService from '../services/ResourceService'
    import PriceService from '../services/PriceService'
    import PluginRepository from '../plugins/PluginRepository'
    import TransferService from '../services/TransferService'
    import ContactService from '../services/ContactService'
    import {Blockchains, BlockchainsArray} from '../models/Blockchains'
    import PopupService from '../services/PopupService'
    import PasswordService from '../services/PasswordService'
    import KeyPairService from '../services/KeyPairService'
    import Keypair from '../models/Keypair'
    import AccountService from '../services/AccountService'
    import {Popup} from '../models/popups/Popup';
    import * as Http from '../util/Http'

    import Eos from 'eosjs';
    import Big from 'big.js';
    const {format} = Eos.modules;
    const encodeName = format.encodeName

    const uniqueToken = x => `${x.account}${x.blockchain}${x.name}${x.symbol}`;

    export default {
        data () {return {
            searchTerms:'',
            Blockchains:Blockchains,
            isSimple:true,
            sending:false,
            showingAll:false,
            captcha: '',
            isValidCaptcha:false,
            invalidCatchaCode:'',
            invalidAccountCode:'',
            isValidAccount:false,
            accMinLen: 5,
            accMaxLen: 11,
            selected:null,
            status: null,

            eos:null,
            eosyName:'',
            regPubKey:null,
            accountExists:false,
            regCatchaCode:'',
            memo:'',
            BankAccountName:'counter.bank',
            SAServerUrl:'https://sa.biteye.pro/',
        }},
        computed:{
            ...mapState([
                'scatter',
            ]),
            ...mapGetters([
                'accounts',
                'contacts',
                'keypairs',
            ]),
            vaultEntries(){
                const terms = this.searchTerms.toLowerCase().trim();
                return this.keypairs.filter(keypair => {
                    if(keypair.name.toLowerCase().indexOf(terms) > -1) return true;
                    if(keypair.accounts().some(x => x.name.toLowerCase().indexOf(terms) > -1)) return true;
                    return false;
                });
            },
            accRegExp () {
                return new RegExp(`^[a-z]{1}[1-5a-z]{${this.accMinLen - 1},${this.accMaxLen}}$`)
            },            
        },
        mounted(){
            this.selected = Keypair.placeholder();
            this.refreshCatcha()
        },
        methods:{             
            back(){	    		
	    	    this.$router.push({name:this.RouteNames.HOME})
            },
            selectKeypair(keypair){
                console.log(keypair);
                this.selected = keypair ? keypair.clone() : null;
                if(this.selected) this.regPubKey = this.selected.publicKeys.find(pk=>pk.blockchain === 'eos').key; 
            },
            async checkCatcha (catcha) {
                this.invalidCatchaCode = 'invalid captcha code';
                const url = `${this.SAServerUrl}index/checkCaptcha`;
                let rs = await Http.post(url, {captchaCode: catcha});
                console.log(rs)
                this.isValidCaptcha = rs.errno === 0;
                if (this.isValidCaptcha) {
                    this.invalidCatchaCode = 'captha code verified';
                } else {
                    this.invalidCatchaCode = 'invalid captcha code';
                }
            }, 
            async refreshCatcha () {
                this.captcha = 'requesting code...'
                const url = `${this.SAServerUrl}index/captcha`
                try {
                    let rs = await Http.get(url)
                    console.log(rs)
                    //rs = rs.data                    
                    if (rs.errno === 0) {
                    this.captcha = rs.data
                    this.regCatchaCode = ''
                    return
                    }
                } catch (error) {
                }
                this.captcha = 'failed to get code...'
            },
            async switchSimple(){                
                this.isSimple = !this.isSimple;
            },
            async checkAccountExists (accountName) {
                const plugin = PluginRepository.plugin(Blockchains.EOSIO);
                const network = await plugin.getEndorsedNetwork();
                const eos = Eos({httpEndpoint:`${network.fullhost()}`, chainId:network.chainId});
                //const eos = Eos({httpEndpoint:`http://192.168.2.106:8888`, chainId:'cf057bbfb72640471fd910bcb67639c22df9f92470936cddc1ade0e2f2e7dc4f'});

                if (accountName.charAt(0) === '.') {
                accountName = accountName.substring(1)
                }
                const lower = encodeName(accountName, false)
                const upper = Big(lower).add(1).toString()
                let options = {
                json: true,
                code: this.BankAccountName,
                scope: this.BankAccountName,
                table: 'assets',
                lower_bound: lower,
                upper_bound: upper,
                limit: 1
                }
                this.accountExists = false;
                await eos.getTableRows(options).then(rs => {                
                    this.accountExists = (rs.rows.length > 0);
                })
            },
            async send(){                
                if(this.sending) return false;
                this.status = 'checking...';    
                if(this.accountExists) return PopupService.push(Popup.prompt("Invalid account name", "Account name is already in used, please choose another one"));
                if(!this.isValidAccount) return PopupService.push(Popup.prompt("Invalid account name", "Account is 5~11 characters long, only contains the characters a-z and 1–5, and start with a letter"));
                if(!this.isValidCaptcha) return PopupService.push(Popup.prompt("Invalid code", "Please check and enter the correct code"));                
                if(this.isSimple){
                    this.regPubKey = null;
                    const vaultName = `VaultEntry-EOSY-${this.eosyName}`;
                    if(this.keypairs.some(keypair => keypair.name == vaultName)) return PopupService.push(Popup.prompt("Invalid vault name", "vault with the EOSY name already exists. Please check it"));    
                    
                    //generate the key pair                    
                    this.selected.name = vaultName;      
                    this.status = 'Generating new Keys';              
                    await KeyPairService.generateKeyPair(this.selected); 
                    this.status = 'A new Key was generated.';  
                    await KeyPairService.makePublicKeys(this.selected);
                    
                    this.regPubKey = this.selected.publicKeys.find(pk=>pk.blockchain === 'eos').key; 
                    console.log(this.regPubKey)
                }else{
                    if(this.regPubKey == null){
                        PopupService.push(Popup.prompt("Invalid key pair", "Please choose the key pair from the key pair list"));
                    }
                }               
                this.status = 'Generating new EOSY Account, please waiting...';  
                this.sending = true;
                //const url = 'https://sa.biteye.pro/figaccount/create'
                const url = `${this.SAServerUrl}figaccount/create`
                let rs
                try {
                    rs = await Http.post(url, {acc: this.eosyName, pk: this.regPubKey, captchaCode: this.regCatchaCode})
                    //rs = rs.data
                    
                } catch (error) {
                }
                 console.log(rs)
                // this.$refs.shareAccountRegister.hide()
                const isSuccess = rs && rs.errno === 0
                if (isSuccess) {                     
                    this.status = 'A new EOSY Account was generated.'; 
                    
                    await KeyPairService.saveKeyPair(this.selected);
                    this.status = 'Linking available accounts. Please wait.';

                    await AccountService.importAllAccounts(this.selected);
                    this.status = 'A new EOSY Account was imported.'; 

                    this.selected = this.keypairs.find(x => x.id === this.selected.id).clone();

                } else {
                    this.isRegistering = false
                    if (rs.errno === 100001) {
                        PopupService.push(Popup.prompt("Invalid check code", "Please double check the check code"));
                    } else {

                        PopupService.push(Popup.prompt("Sign up failed", `There is some problem during signing up, ${rs.errno}:${rs.errmsg}`));
                    }
                }
                this.refreshCatcha();
                PopupService.push(Popup.prompt("Sign up sucessfuly", `Enjoy the EOS journey with your EOSY account`));
                this.back();
                this.status = null;
                this.sending = false;
            },
        },
        watch:{            
            regCatchaCode: function (newVal, oldVal) {
                if (newVal.length < 4) {
                    this.isValidCaptcha = false;
                    this.invalidCatchaCode = 'please enter the code';
                } else {
                    this.checkCatcha(newVal);
                }
            },
            eosyName: async function (newVal, oldVal) {                
                if(!this.accRegExp.test(this.eosyName)){
                    this.isValidAccount = false;
                    this.invalidAccountCode = "Account is 5~11 characters long, only contains the characters a-z and 1–5, and start with a letter";
                }else{
                    await this.checkAccountExists(this.eosyName);
                    if(this.accountExists){
                        this.isValidAccount = false;
                        this.invalidAccountCode = 'account name already exsists, please choose another one';
                    }else{
                        this.isValidAccount = true;
                        this.invalidAccountCode = "Account name is OK.";
                    }                    
                }
            },
        }
    }
</script>

<style scoped lang="scss" rel="stylesheet/scss">
    @import "../_variables";

    .transfer {
        flex:1;
        display:flex;
        flex-direction: row;
        background:rgba(255,255,255,0.5);

        .address-book {
            flex:1;
            display:flex;
            flex-direction: column;
            background:$light-blue;
            position: relative;
            z-index:2;

            .bg {
                position:absolute;
                top:10px; bottom:0; left:0; right:0;
                background:#fff;
                z-index:-1;
            }

            .head {
                padding:20px 30px;
                background:#fff;
                font-size: 16px;
                font-weight: 600;
                color:$medium-grey;
                border-top-right-radius:8px;
                box-shadow:10px -10px 20px rgba(0,0,0,0.01);
            }

            .search {
                flex:0 0 auto;
                padding:0 30px;
                overflow: hidden;
                height:40px;
                background:#f3f3f3;
                border-top:1px solid #e1e1e1;
                border-bottom:1px solid #e1e1e1;

                .input {

                    .icon {
                        float:left;
                        font-size: 13px;
                        line-height:40px;
                        color: #aeaeae;
                    }

                    input {
                        margin-left:10px;
                        font-size: 11px;
                        float:left;
                        outline:0;
                        border:0;
                        height:40px;
                        background:transparent;

                    }
                }

            }
            .scroller {
                flex:1;
                display:flex;
                flex-direction: column;
                overflow-y:auto;
                overflow-x:hidden;
            }

            .accounts {
                flex:0 0 auto;
                /*overflow-y:auto;*/
                max-height:1200px;
                transition:max-height 0.5s ease;


                &.hidden {
                    max-height:0;
                    overflow:hidden;
                }

                .account {
                    flex:1 0 auto;
                    padding:15px 30px;
                    background:transparent;
                    transition:all 0.2s ease;
                    transition-property: background, padding;
                    overflow: hidden;
                    display:flex;
                    justify-content: center;
                    align-items: center;

                    &:not(.static){
                        cursor: pointer;
                    }

                    &.copy {
                        cursor: grabbing;
                    }

                    &:not(:last-child){
                        border-bottom:1px solid rgba(0,0,0,0.05);
                    }

                    .info {
                        width:calc(100% - 20px);
                        float:left;

                        .name {
                            font-size: 16px;
                            font-weight: 500;
                        }

                        .description {
                            margin-top:3px;
                            font-size: 11px;

                            &.linked-account {
                                font-size: 9px;
                                font-weight: bold;
                                color:$mid-dark-grey;
                                margin-bottom:5px;
                            }

                            i {
                                color:$dark-grey;
                                margin-right:3px;
                            }
                        }
                    }

                    .icon {
                        width:20px;
                        float:left;
                        font-size: 18px;
                        color:rgba(0,0,0,0.1);
                        text-align:right;

                        transition: color 0.5s ease;
                    }

                    &:hover {
                        &:not(.static){
                            background:rgba(0,0,0,0.015);
                            padding:15px 35px;

                            .icon {
                                color:rgba(0,0,0,0.4);
                            }
                        }
                    }
                }
            }

            .addresses {
                display:flex;
                flex-direction: column;
                overflow-y:auto;
                overflow-x:hidden;
                background:rgba(0,0,0,0.02);

                .address {
                    height:40px;
                    line-height:40px;
                    padding:0 10px 0 30px;
                    border-bottom:1px solid rgba(0,0,0,0.05);
                    background:rgba(0,0,0,0);
                    transition: all 0.2s ease;
                    transition-property: background, padding;
                    cursor: pointer;
                    font-size: 13px;
                    overflow: hidden;

                    &:hover {
                        background:#fff;
                        padding-left:35px;
                    }

                    .name {
                        width:calc(100% - 25px);
                        float:left;
                    }

                    .remove {
                        width:25px;
                        height:25px;
                        line-height:24px;
                        margin-top:6px;
                        float:left;
                        text-align:center;
                        border:1px solid rgba(0,0,0,0.1);
                        border-radius: 2px;
                        color:rgba(0,0,0,0.2);
                        transition: all 0.2s ease;
                        transition-property: background, border, color;

                        &:hover {
                            border:1px solid $red;
                            background:$red;
                            color:#fff;
                        }
                    }
                }
            }
        }

        .details {
            float:left;
            flex:2.5;
            display:flex;
            flex-direction: column;
            box-shadow: inset 1px 0 3px rgba(0,0,0,0.1);

            .actions {
                flex:0 0 auto;
                height:80px;
                display: flex;
                justify-content: space-between;
                background:$light-blue;
                padding:15px 50px 0 30px;
                float:left;
                width:100%;

                .action {
                    margin-left:10px;
                    cursor: pointer;
                    border-radius: 2px;
                    border:1px solid #fff;
                    height:50px;
                    line-height:48px;
                    padding:0 20px;
                    text-align:center;
                    font-size: 24px;
                    color:#fff;

                    transition:all 0.2s ease;
                    transition-property: background, border, color;

                    &:hover, &.active {
                        background:#fff;
                        border:1px solid #fff;
                        color:$light-blue;
                    }
                }
            }

            .data {
                flex:1;
                padding:40px;
                overflow-y:auto;
                overflow-x:hidden;

                .breaker {
                    width:100%;
                    clear:both;
                }
                .status {
                    .title {
                        color:$red;
                        animation: pulsate 1s ease-out;
                        animation-iteration-count: infinite;
                    }

                    .description {
                        color:$black;
                    }
                }

                .inline-inputs {
                    display:flex;

                }

                .inputs {
                    margin-bottom:20px;
                    position: relative;

                    label {
                        font-size: 11px;
                    }

                    .action {
                        cursor: pointer;
                        position:absolute;
                        top:15px;
                        right:0;
                        width:35px;
                        height:35px;
                        line-height:33px;
                        font-size: 16px;
                        text-align:center;
                        border:1px solid rgba(0,0,0,0.2);
                        color:rgba(0,0,0,0.3);
                        border-radius:2px;

                        transition: all 0.2s ease;
                        transition-property: color, background, border;

                        &:hover {
                            background:$light-blue;
                            border:1px solid $light-blue;
                            color:#fff;
                        }
                    }

                    .prefix {
                        position:absolute;
                        bottom:3px;
                        left:0;
                        font-size: 22px;
                        color:$medium-grey;
                        font-weight: bold;


                        &.icon {
                            cursor: pointer;

                            &:hover {
                                color:$red;
                            }
                        }
                    }

                    input {
                        width:100%;
                        border:0;
                        outline:0;
                        background:transparent;
                        border-bottom:1px dashed rgba(0,0,0,0.2);
                        font-size: 24px;
                        margin-top:10px;

                        transition:all 0.4s ease;
                        transition-property: padding;

                        &.with-action {
                            padding-right:40px;
                        }

                        &.with-prefix {
                            padding-left:17px;
                        }

                        &.with-icon {
                            padding-left:25px;
                        }

                        $placeholdercolor:rgba(0,0,0,0.3);
                        &::-webkit-input-placeholder { color: $placeholdercolor; }
                        &::-moz-placeholder { color: $placeholdercolor; }
                        &:-ms-input-placeholder { color: $placeholdercolor; }
                        &:-moz-placeholder { color: $placeholdercolor; }
                    }

                    &.half {
                        flex:2;

                        &:nth-child(2){
                            margin-left:20px;
                        }
                    }

                    &.third {
                        flex:1;

                        &:nth-child(2){
                            margin-left:20px;
                        }
                    }
                }
            }
        }

    }



</style>
