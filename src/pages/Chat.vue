<template>
    <div>
        <div class="layout">
            <Sidebar />
            <div class="container">
            <div class="content-container">
                    <ConversasComponent
                        :chats="getMyChats"
                        :choosedContact="choosedContact"
                        :profilePic="getProfilePic(choosedContact)"
                        :nameContact="getContactName(choosedContact)"
                        @setChats="setChats"
                        @onClickContact="onClickContact"
                    />

                    <div class="chat-container" >
                        <header class="header-contact" v-if="(choosedContact.length != 0)">
                            <div class="container-info-ctt">
                                <img :src="getProfilePic(choosedContact)" :alt="getContactName(choosedContact)"/>
                                <h3 v-html="getContactName(choosedContact)"></h3>
                            </div>
                            <div class="close" @click="choosedContact = ''; messages = ''"><span class="material-icons">close</span></div>
                        </header>

                        <ul style="overflow-x: hidden">
                            <button class="load-more" @click="loadMore"  v-if="(messages.length > 0)">
                                Load more messages
                            </button>
                            <div class="waiting-container" v-if="(messages.length == 0)">
                                <img :src="ImageLoader" alt="Smartphone" />
                                <h2> {{i18n.chooseContact}}</h2>
                                
                            </div>
                            <div v-if="messages.length > 0">
                                <li v-bind:key="message.id" v-for="message in messages" :id="message.id">
                                    <MessageComponent
                                    :isMe="message.fromMe ? true : false"
                                    :isWarning="!message?.body && message.type !== 'chat' && !['ptt', 'audio'].includes(message.type)"
                                    
                                    :session="getSession"
                                    :token="getToken"
                                    :message="message"
                                    :profilePic="getProfilePic(choosedContact)"
                                    :nameContact="getContactName(choosedContact)"
                                    @selectMessageId="setSelectedMessage"
                                    />
                                </li>
                            </div>
                            <div id="messagesEnd" ref="messagesEnd" />
                        </ul>
                        <div class="reply-container" v-if="selectedMessage.length != 0">
                            <div class="content">
                                <MessageComponent
                                :isMe="selectedMessage.fromMe ? true : false"
                                :isWarning="!selectedMessage?.body && selectedMessage.type !== 'chat' && !['ptt', 'audio'].includes(selectedMessage.type)"
                                
                                :session="getSession"
                                :token="getToken"
                                :message="selectedMessage"
                                @selectMessageId="()=> {}"
                                />
                            </div>
                            <div>
                                <div class="my-tooltip" name="Cancelar" @click="selectedMessage = ''"><span class="material-icons">close</span></div>
                            </div>
                        </div>
                        <div class="bottom-container" v-if="(choosedContact.length != 0)">
                            <textarea placeholder="Digite uma mensagem..." id="message" @keydown.enter.prevent="sendMessage" v-model="data.message"/>

                            <div class="action-buttons">
                                <div>
                                    <div v-if="emoji == true">
                                        <Picker :data="emojiIndex" :showPreview="false" :showSearch="false" set="apple" @select="addEmoji" />
                                    </div>
                                    <button @click="tooggleEmojiBox" v-if="emoji">
                                        <span class="material-icons">close</span>
                                    </button>
                                    <button @click="tooggleEmojiBox" v-else>
                                        <span class="material-icons">add_reaction</span>
                                    </button>

                                    <label>
                                    <input type="file" @change="onChangeAttach" />
                                    <div class="attach-info">
                                        <span class="material-icons">attach_file</span>
                                    </div>
                                    </label>

                                    <label>
                                    <div class="attach-info" data-bs-toggle="modal" data-bs-target="#sendOptModal">
                                        <span class="material-icons">list</span>
                                    </div>
                                    </label>
                                </div>
                                    <span class="material-icons" style="cursor:pointer;" id="startRecording" @click="startRecording" v-if="!recordState">mic</span>
                                    <div class="contador" v-else>
                                        <div class="main-cont">
                                            <span class="material-icons" style="cursor:pointer;"  v-if="recordState" @click="cancelRecording">cancel</span>
                                        <div class="counter">
                                            <p v-html="getTimeRecording" v-if="recordState"></p>
                                        </div>
                                        <span class="material-icons" style="cursor:pointer;"  v-if="recordState" @click="finishRecording">stop</span>
                                        </div>
                                    </div>
                                    <span class="material-icons" style="cursor:pointer;"  @click="sendMessage">send</span>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
        <!-- Modal -->
        <div class="modal fade" id="sendOptModal" tabindex="-1" aria-labelledby="sendOptModalLabel" aria-hidden="true">
        <div class="modal-dialog">
            <div class="modal-content">
            <div class="modal-header">
                <h5 class="modal-title" id="sendOptModalLabel" >Envio de listas e botões</h5>
                <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
            </div>
            <div class="modal-body">
                <h4>Botões</h4>
                <div class="" style="cursor:pointer;" v-for="(btn,index) in buttons" @click="sendButton(btn)" v-bind:key="index">
                    <h6>{{btn.name}}</h6>
                    <small class="form-text text-muted">{{btn.message}}</small>
                    <hr/>
                </div>
                <h4>Listas</h4>
                <div class="" style="cursor:pointer;" v-for="(lst,index) in lists" @click="sendList(lst)" v-bind:key="index">
                    <h6>{{lst.name}}</h6>
                    <small class="form-text text-muted">{{lst.title}}</small>
                    <hr/>
                </div>
            </div>
            <div class="modal-footer">
            </div>
            </div>
        </div>
        </div>
    </div>
</template>
<script>
import { config } from '../config';
import configHeader from "../util/sessionHeader";
import api, {socket} from '../services/api.js'
import {getSession, getToken} from '../services/auth'
import {getConfigs} from '../services/settings'
import {getButtons} from '../services/settings_buttons'
import {getLists} from '../services/settings_lists'
import router from '../router/index'
import {useStore} from '../stores/dataStore'
import data from "emoji-mart-vue-fast/data/all.json";
import "emoji-mart-vue-fast/css/emoji-mart.css";
import { Picker, EmojiIndex } from "emoji-mart-vue-fast/src";
let emojiIndex = new EmojiIndex(data);
import MicRecorder from "mic-recorder-to-mp3";
import {language} from '../services/language'

//Components
import MessageComponent from '../components/MessageComponent.vue'
import ConversasComponent from '../components/ConversasComponent.vue'
import Sidebar from '../components/Sidebar.vue'




//Assets
import ImageLoader from '../assets/ic_loader_chat.svg'
const defaultImage = "https://i.pinimg.com/736x/51/24/9f/51249f0c2caed9e7c06e4a5453c57857.jpg";

    export default {        
        components: {
            MessageComponent,
            ConversasComponent,
            Picker,
            Sidebar,
        },
        async mounted(){
            const onLoad = async () => {
                await this.checkConnection();
            }
            const addMessage = async (message) => {
                await this.addMessage(message)
            }
            onLoad()
            socket.on("received-message", (message) => {
                addMessage(message?.response || message)
            });
            const incomingCallEvents = async(call) =>{
                await this.incomingCallEvents(call)
            }
            socket.on("incomingcall", (call) => {
                incomingCallEvents(call)
            });

        },
        setup(){
            const data = useStore()

            return { data }
        },
        data() {
            return {
                choosedContact: '',
                messages: [],
                selectedMessage: [],
                emoji: false,
                emojiIndex: emojiIndex,
                emojisOutput: "",

                recordState: false,
                IsBlocked: null,
                stop: null,
                segundos: 0,
                minutos: 0,
                recorder: new MicRecorder({bitRate: 128}),
                intervalId: null,

                loadingMoreMessages: false,
                hasNoMore: false,

                settings: getConfigs(),
                buttons: getButtons(),
                lists: getLists(),
                i18n: language(),

            }
        },
        methods: {
            async addMessage(message){
                this.data.chats.map((chat)=>{
                    let idContact = chat.id._serialized ? chat.id._serialized : chat.id;
                    if(idContact == message.chatId){
                        var exist = false;
                        chat.msgs.map(msg=>{
                            if(msg.id == message.id){
                                exist = true;
                            }
                        })
                        if(!exist){
                            chat.msgs.push(message)
                            chat.timestamp = message.timestamp;
                            chat.unreadCount = chat.unreadCount+1;
                        }
                        if(message.fromMe){ chat.unreadCount = 0 }
                    }
                })
                if(this.choosedContact){
                    let idContact = this.choosedContact.id._serialized ? this.choosedContact.id._serialized : this.choosedContact.id;
                    if(idContact == message.chatId){
                        this.scrollToBottom();
                    }
                }
            },
            async checkConnection() {
                try {
                    this.$swal({
                        title: 'Please, wait...',
                        timerProgressBar: true,
                        showConfirmButton: false,
                        didOpen: () => {
                            this.$swal.showLoading()
                        }})
                    await api.get(`${getSession()}/check-connection-session`, configHeader());
                    await this.getAllContacts()
                    .then(async ()=>{
                        await this.getAllChats();
                    })
                    this.$swal().close()
                } catch (e) {
                    router.push('/login')
                    console.log(e)
                    if(e.response){
                        if(e.response.data) {
                            this.$swal(e.response.data.message)
                        }else{
                            this.$swal(e.message)
                        }
                    }
                }
            },
            async getAllContacts() {
                return new Promise(async (resolve, reject) => {
                    const {data} = await api.get(`${getSession()}/all-contacts`, configHeader());
                    const arr = [];

                    for (const contact of data.response) {
                        if (contact.isMyContact && contact.id.user !== undefined)
                            arr.push(contact);
                    }
                    this.data.contacts = arr;
                    resolve();
                });
            },
            async getAllChats() {
                try {
                    const {data: {response}} = await api.get(`${getSession()}/all-chats-with-messages`, configHeader());

                    const arr = [];
                    for (const elem of response) {
                        if (!elem.archive) {
                            var newarray = elem
                            if(elem.id.includes('@g.us')){
                                newarray = elem;
                                if(newarray.msgs.length == 0){ newarray.msgs = {id: 0, t: 31546800, body: 'No messages'}}
                            }else{
                                this.data.contacts.map((contact)=>{
                                    if(contact.id._serialized == elem.id){
                                        newarray = contact;
                                        newarray.msgs = elem.msgs;
                                        if(newarray.msgs.length == 0){ newarray.msgs = {id: 0, t: 31546800, body: 'No messages'}}
                                        newarray.unreadCount = elem.unreadCount;
                                    }
                                })
                            }
                            arr.push(newarray);
                        }
                    }
                    this.data.chats = arr;
                    this.data.dados = arr;
                } catch (e) {
                    console.log(e)
                    /*
                    const {data: {response}} = await api.get(`${getSession()}/all-chats-with-messages`, configHeader());

                    const arr = [];
                    for (const elem of response) {
                        if (!elem.archive) {
                            var newarray = elem
                            if(elem.id.includes('@g.us')){
                                newarray = elem;
                                if(elem.msgs.length < 1){
                                    newarray.msgs = [{id: 0, t: 0, body: 'No messages'}]
                                }
                            }else{
                                this.data.contacts.map((contact)=>{
                                    if(contact.id._serialized == elem.id){
                                        newarray = contact;
                                        newarray.msgs = elem.msgs;
                                        if(elem.msgs.length < 1){
                                            newarray.msgs = [{id: 0, t: 0, body: 'No messages'}]
                                        }
                                        newarray.unreadCount = elem.unreadCount;
                                    }
                                })
                            }
                            arr.push(newarray);
                        }
                    }
                    this.data.chats = arr;
                    this.data.dados = arr;*/
                }
            },
            async setChats(chats){
                this.chats = chats;
            },
            async sendMessage(){
                var nMessage = this.data.message;
                this.data.message = ''
                if (!!nMessage.trim() && !!this.getSession()) {
                    var by = '';
                    if(this.settings.attendantName) {by = '*'+this.settings.attendantName+':*\n'}
                    let idContact = this.choosedContact.id._serialized ? this.choosedContact.id._serialized : this.choosedContact.id;
                    let endpoint = "send-message";

                    const body = {
                        phone: idContact.replace(/[@c.us,@g.us]/g, ""),
                        message: by + nMessage,
                    };

                    if (idContact.includes("@g.us")) {
                        body.isGroup = true;
                    }

                    if (this.selectedMessage.length != 0) {
                        body.messageId = this.selectedMessage.id;
                        endpoint = "send-reply";
                    }

                    await api.post(`${this.getSession()}/${endpoint}`, body, configHeader());
                    //await this.onClickContact(this.choosedContact)
                    this.selectedMessage = []
                } else {
                    this.$swal('Digite uma mensagem!')
                }
            },
            async sendButton(btn){
                let idContact = this.choosedContact.id._serialized ? this.choosedContact.id._serialized : this.choosedContact.id;
                var body = {
                    "isGroup": false,
                    "phone": idContact.replace('@c.us','@g.us',''),
                    "message": btn.message,
                    "options": {
                        "useTemplateButtons": "true",
                        "buttons": btn.buttons,
                        "title": btn.title,
                        "footer": btn.footer
                    }
                }
                if (idContact.includes("@g.us")) {
                    body.isGroup = true;
                }
                await api.post(`${getSession()}/send-buttons`, body, configHeader());
                $('#sendOptModal').modal('hide')
            },
            async sendList(list){
                let idContact = this.choosedContact.id._serialized ? this.choosedContact.id._serialized : this.choosedContact.id;
                var body = {
                    "isGroup": false,
                    "phone": idContact.replace('@c.us','@g.us',''),
                    "buttonText": list.buttonText,
                    "description": list.description,
                    "sections": list.sections,
                }
                if (idContact.includes("@g.us")) {
                    body.isGroup = true;
                }
                await api.post(`${getSession()}/send-buttons-list`, body, configHeader());
                $('#sendOptModal').modal('hide')
            },
            scrollToBottom() {
                setTimeout(() => {
                    if (this.$refs.messagesEnd !== null) {
                        this.$refs.messagesEnd.scrollIntoView({behavior: "smooth"});
                    }
                },300)
            },
            async onClickContact(contact){
                this.choosedContact = contact;
                this.$swal({
                    title: 'Please, wait...',
                    timerProgressBar: true,
                    showConfirmButton: false,
                })
                let idContact = contact.id._serialized ? contact.id._serialized : contact.id;

                try {
                        const {data} = await api.get(`${getSession()}/get-messages/${idContact}`, configHeader());
                        //if(this.settings.sendSeen) {await api.post(`${getSession()}/send-seen`,{phone: idContact.replace("@c.us", "")}, configHeader());}
                        contact.unreadCount = 0;
                        console.log(data);
                        this.messages = data?.response || data || [];
                        this.choosedContact.msgs = data?.response || data || [];
                        this.$swal().close()
                        this.scrollToBottom()
                } catch (e) {
                    if(e){
                        console.log(e)
                        this.$swal(e)
                    }
                }
            },
            async loadMore() {
                this.loadingMoreMessages = true;
                try {
                    let id = this.messages[0].id
                    let idContact = this.choosedContact.id._serialized ? this.choosedContact.id._serialized : this.choosedContact.id;
                    let param = `?isGroup=false&id=${id}`;
                    if (idContact.includes("@g.us")) {
                        param = `?isGroup=true&id=${id}`;
                    }
                    const { data } = await api.get(`${getSession()}/get-messages/${idContact}${param}`, configHeader());
                    if (data && data.response && Array.isArray(data.response)) {
                        this.messages = [...data.response, ...this.messages]
                    }
                    if (data && !data.response) {
                        this.hasNoMore = true;
                    }
                } catch (e) {
                    this.$swal(e.message)
                    console.log(e);
                } finally {
                    this.loadingMoreMessages = false;
                }
            },
            onChangeAttach(e) {
                if (e.target.files && e.target.files[0]) {
                    const reader = new FileReader();
                    const filename = e.target.files[0].name;
                    reader.readAsDataURL(e.target.files[0]);

                    const that = this;
                    reader.onload = async function (e) {
                    let idContact = that.choosedContact.id._serialized ? that.choosedContact.id._serialized : that.choosedContact.id;
                        const base64 = e.target.result;
                        var options = {
                            base64: base64,
                            phone: idContact.replace(/[@c.us,@g.us]/g, ""),
                            message: "",
                            filename: filename,
                            isGroup: false,
                        };
                        if (idContact.includes("@g.us")) {
                            options.isGroup = true;
                        }
                        try {
                            await api.post(`${getSession()}/send-file-base64`, options, configHeader());
                        } catch (error) {
                            console.log("Catch onChangeAttach()", error);
                        }
                    };
                }
            },
            

            /** FUNÇÕES PARA GRAVAÇÃO DE AUDIO E CONTAGEM DE TEMPO - INICIO */
            tooggleCount(){
                    if (this.stop === false) {
                        this.intervalId = setInterval(() => {
                                if (this.segundos >= 59) {
                                    this.zerar();
                                    this.incrementarMinuto();
                                }

                                this.segundos = this.segundos + 1;
                        }, 1000);
                    }else{
                        clearInterval(this.intervalId)
                        this.intervalId = null
                    }
            },
            zerarCronometro() {
                this.segundos = 0;
                this.minutos = 0;
                this.tooggleCount();
            },

            startRecording() {
                navigator.mediaDevices.getUserMedia({audio: true},
                    () => {
                        // alert("Permission Granted");
                        this.IsBlocked = false;
                    },
                    () => {
                        alert("Permission Denied");
                        this.IsBlocked = true;
                    },
                );
                if (this.isBlocked) {
                    alert("Permission Denied");
                } else {
                    this.recorder.start().then(() => {
                        this.recordState = true;
                        this.stop = false;
                        this.tooggleCount()
                    }).catch((e) => {
                        console.error(e);
                    });
                }
            },

            cancelRecording() {
                this.recorder.stop().getMp3().then(([buffer, blob]) => {
                }).catch((e) => {
                    console.log(e);
                });
                this.recordState = null;

                this.stop = true;
                this.zerarCronometro();
            },

            finishRecording() {
                this.recordState = null
                this.stop = true;
                this.zerarCronometro();
                this.tooggleCount()

                this.recorder.stop().getMp3().then(([buffer, blob]) => {
                    const reader = new FileReader();
                    reader.readAsDataURL(blob);
                    var that = this;
                    reader.onloadend = async function () {
                        const base64data = reader.result;
                        let idContact = that.choosedContact.id._serialized ? that.choosedContact.id._serialized : that.choosedContact.id;
                        await api.post(`${that.getSession()}/send-voice-base64`, {
                            base64Ptt: base64data,
                            phone: idContact,
                        }, configHeader());
                    };

                    const file = new File(buffer, "audio.mp3", {
                        type: blob.type,
                        lastModified: Date.now()
                    });
                    new Audio(URL.createObjectURL(file));

                }).catch((e) => {
                    alert("We could not retrieve your message");
                    console.log(e);
                });
            },

            incrementarMinuto() {
                this.minutos = this.minutos+1;
            },

            zerar() {
                this.segundos = 0;
            },
            /** FUNÇÕES PARA GRAVAÇÃO DE AUDIO E CONTAGEM DE TEMPO - FIM */

            async incomingCallEvents(call){   
                if(this.settings.rejectCall){ await api.post(`${getSession()}/reject-call`,{callId: call.id}, configHeader());}
                if(this.settings.rejectCall && this.settings.msgRejectCall){ await api.post(`${getSession()}/send-message`,{phone: call.peerJid.replace(/[@c.us,@g.us]/g, ""), message: this.settings.msgRejectCall}, configHeader());}
                if(!this.settings.rejectCall){this.$swal('O telefone '+call.peerJid.replace(/[@c.us,@g.us]/g, "")+' está te ligando...')}
            },
            setSelectedMessage(message){
                this.selectedMessage = message;
            },
            getContactName(contact){
                if(!contact.name){
                    return contact?.id?.replace("@c.us", "").replace("@g.us", "")
                }else{
                    return contact.name
                }                
            },
            getPhoneNumber(contact){
                if(contact.id._serialized){
                    if(contact.id._serialized.includes('@g.us')){
                        return '';
                    }else{
                        return this.maskPhone(contact.id._serialized.replace('55','@g.us','@c.us',''))
                    }
                }else if(contact.id){
                    if(contact.id.includes('@g.us')){
                        return '';
                    }else{
                        return this.maskPhone(contact.id.replace('55','@g.us','@c.us',''))
                    }
                }else{
                    return '';
                }
            },
            maskPhone( value ) {
            return value
                .replace(/\D/g, "")
                .replace(/(\d{2})(\d)/, "($1) $2")
                .replace(/(\d{5})(\d{4})(\d)/, "$1-$2");
            },

            getProfilePic(contact){
                if(contact.profilePicThumbObj){
                    if(contact.profilePicThumbObj.eurl){
                        return contact.profilePicThumbObj.eurl
                    }else{
                        return defaultImage;
                    }
                }else{
                    return defaultImage;
                }
            },
            tooggleEmojiBox(){
                this.emoji = !this.emoji;
                document.getElementById('message').focus()
            },
            addEmoji(emoji){
                this.data.message = this.data.message + emoji.native;
                document.getElementById('message').focus()
            },

            //Funções que serão chamadas apenas por retorno
            getSession(){
                return getSession()
            },
            getToken(){
                return getToken()
            },

        },
        computed:{
            getTimeRecording(){
                if(this.minutos === 0){
                    return `${this.segundos}s`
                }else{
                    return `${this.minutos}m ${this.segundos}s`
                }                       
            },
            //Funções que serão chamadas apenas por retorno
            ImageLoader(){
                return ImageLoader;
            },
            getMyChats(){
                const chatsSort = this.data.chats.sort(function(x, y){return y.msgs[y.msgs.length-1]?.timestamp - x.msgs[x.msgs.length-1]?.timestamp;})
                return chatsSort.filter(chat => chat.msgs.length > 0)
            },
        },
 
    }
</script>
<style scoped>
.layout{
  height: 100vh;
  width: 100%;

  display: flex;
  overflow: hidden;

  justify-content: center;
  align-items: center;
  margin-left:43px;
}
.container{
  height: 100vh;
  width: 100%;

  display: flex;
  flex-direction: column;

  justify-content: center;
  align-items: center;
}

.content-container{
    width: 110%;
    height: 100%;
    display: flex;
    overflow: hidden;
    clear: both;
    max-height: 100vh;
}

.emoji-mart-bar {
border: 0;
}

.emoji-mart-anchors {
padding: 0;
border: 0;
}

.emoji-mart-anchor-icon {
color: #a3a3a3;
}

.emoji-mart-anchor-selected {
color: red !important;
}

.emoji-mart-anchor-bar {
background-color: #36aa9f !important;
}

.emoji-mart-anchor-icon {
color: #8b8b8b !important;
}

.emoji-mart-search {
margin: 5px 8px 15px 8px;
}

.emoji-mart-search-icon {
display: none;
}

.emoji-mart-search input {
font-size: 16px;
display: block;
width: 100%;
padding: 10px;
border-radius: 5px;
border: 0px solid #d9d9d9;
outline: 0;
background-color: #e6e6e6;
color: #4a4a4a;
}

.emoji-mart-category-label span {
display: block;
width: 100%;
font-weight: 500;
padding: 5px 6px;
color: #b4b4b4;
background-color: #f0f0f0;
}

.emoji-mart-scroll {
height: 200px;
}

.emoji-mart-bar:last-child {
display: none;
}

.emoji-mart {
    position: absolute !important;
    bottom: 140px;
    width: 100% !important;
    height: 40vh;
    z-index:2;
}

.emoji-mart-category-list {
display: flex;
flex-wrap: wrap;
}
.top-container{  
    display: flex;
  align-items: center;

  justify-content: space-between;
  width: 100%;
}
.top-container span {
display: flex;
align-items: center;
cursor: pointer;
transition-duration: 200ms;
}
.top-container span p {
      border-bottom: 1px solid #000;
}

.top-container span svg {
    margin-right: 10px;
}

.top-container span svg:hover {
    transform: scale(1.1);
}
form.sessions-container{
  display: flex;
  flex-direction: column;
  background: #fff;
  padding: 2em;

  height: 100%;
  width: 20%;
  min-width: 20%;
  position: relative;
  overflow: auto;
}
  form.sessions-container .plus-button {
    position: absolute;
    bottom: 30px;
    right: 30px;
    box-shadow: rgba(0, 0, 0, 0.15) 0px 15px 25px,
    rgba(0, 0, 0, 0.05) 0px 5px 10px;

    border-radius: 50%;
    background: #007af3;
    width: 50px;
    height: 50px;
    cursor: pointer;
    transition-duration: 200ms;

    display: flex;
    align-items: center;
    justify-content: center;
  }
form.sessions-container .plus-button svg {
      color: #fff;
      width: 30px;
      height: 30px;
    }
form.sessions-container .plus-button svg:hover {
      transform: scale(1.05);
      background: #1065ba;
    }
  form.sessions-container ul {
    width: 100%;
    display: flex;
    align-items: center;
    flex-direction: column;
    list-style-type: none;
    }
form.sessions-container ul li {
    margin-top: 1em;
    width: 100%;
}
form.sessions-container ul li label .info-session {
    display: flex;
    flex-direction: column;
    cursor: pointer;

    padding: 20px 10px;
    border-radius: 7px;

    border: 1px solid #f4f6f9;
    transition-duration: 200ms;
}
form.sessions-container ul li label small {
color: #999;
}

form.sessions-container ul li label p {
font-weight: 600;
}
form.sessions-container ul li label input[type="radio"] {
          display: none;
        }
form.sessions-container ul li label input[type="radio"]:checked + .info-session {
    background: #f4f6f9;
}
form.sessions-container ul li label .info-session:hover {
transform: scale(1.03);
background: rgb(240, 255, 253);
}
.header-contact{
    display: flex;
  padding: 5px;
  border-bottom: 1px solid purple;
  background: rgb(66, 66, 66);
}
.header-contact .container-info-ctt {
    display: flex;
    align-items: center;
    width: 100%;
    padding: 0 2em;
}
.header-contact .container-info-ctt img {
    width: 40px;
    height: 40px;
    border-radius: 50%;
    object-fit: cover;
    margin-right: 15px;
}

.header-contact .container-info-ctt h3 {
    font-weight: 500;
    color: rgb(255, 255, 255);
}
.header-contact .container-info-ctt div h6 {
    font-weight: 400;
    font-size: 7pt;
    color: rgb(145, 145, 145);
    position: relative;
    left: 0;
}
.header-contact .close{
    color:white;
    margin-top:15px;
    cursor: pointer;
}
.chat-container{
    display: flex;
    flex-direction: column;
    background: rgba(255, 195, 255, 0.192);
    border-radius: 3px;
    width: 100%;
    height: 100%;
    position: relative;
    z-index: 1;
}
.chat-container h3 {
font-size: 1rem;
}
.chat-container .bottom-container {
    position: relative;
    display: flex;
    flex-direction: column;
    align-items: center;

    width: 100%;
    padding: 15px;
    background: rgb(206, 206, 206);
}
.chat-container .bottom-container .action-buttons {
    width: 100%;
    display: flex;
    align-items: center;
    justify-content: space-between;
}
.chat-container .bottom-container .action-buttons button {
    background: transparent;
    border: 0;
    outline: 0;
}

.chat-container .bottom-container .action-buttons div {
display: flex;
}
.chat-container .bottom-container .action-buttons div button .material-icons,
.chat-container .bottom-container .action-buttons div .material-icons {
    cursor: pointer;
}
.chat-container .bottom-container label {
    display: flex;
    align-items: center;
}
.chat-container .bottom-container label input[type="file"] {
    display: none;
}
.chat-container .bottom-container input {
    width: 100%;
    padding: 10px 15px;
    border-radius: 20px;
    border: 0;
    outline: 0;
}
.chat-container .bottom-container svg {
    margin-left: 20px;
    cursor: pointer;
    color: #666;
    transition-duration: 200ms;
}
.chat-container .bottom-container svg:nth-child(1) {
    margin-left: 10px;
    margin-right: 10px;
    }

    .chat-container .bottom-container svg:hover {
    color: #000;
    }
.chat-container ul {
height: 100%;
overflow: auto;
padding: 2em;
list-style-type: none;
}
.chat-container ul::-webkit-scrollbar {
    width: 7px;
    height: 7px;
}

.chat-container ul::-webkit-scrollbar-track {
background: #f1f1f1;
}

.chat-container ul::-webkit-scrollbar-thumb {
background: #9a9a9a;
transition-duration: 200ms;
border-radius: 10px;
}

.chat-container ul::-webkit-scrollbar-thumb:hover {
background: #a5a5a5;
}

.chat-container ul li {
display: flex;
margin-bottom: 10px;
}
.chat-container textarea {
    width: 100%;
    padding: 10px;
    border-radius: 5px;
    outline: none;
    height: 70px;
    font-size: 16px;
    background: rgb(228, 228, 228);
    color: rgb(36, 36, 36);
    border: 2px solid rgb(99, 8, 84);
    margin-bottom: 10px;
    transition-duration: 200ms;
}
.chat-container textarea :focus {
    border: 2px solid green;
}

.waiting-container{
    color: #f5f5f5;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-direction: column;
    text-align: center;

    height: 100%;
}
.waiting-container img {
    width: 100px;
    height: 100px;
    margin-bottom: 2em;
}

.waiting-container h2 {
    font-size: 1.3rem;
    color:purple;
}

.waiting-container p {
    margin-top: 0.5em;
    width: 350px;
    font-size: 1rem;
}

@keyframes pulsate-bck {
    0% {
        transform: scale(1);
    }
    50% {
        transform: scale(0.9);
    }
    100% {
        transform: scale(1);
    }
}
.contador{
    display: block;
    width: 160px;
    min-width: 160px;
}

.contador .main-cont {
    display: flex;
    justify-content: flex-end;
    align-items: center;
}
.contador .main-cont svg {
    cursor: pointer;
    width: 26px;
    height: 26px;
}
.contador .main-cont svg:nth-child(1) {
    color: #c25252;
}

.contador .main-cont svg:nth-child(3) {
    color: #569241;
}
.contador .main-cont .counter p {
    font-size: 16px;
    text-align: center;
}
.load-more{
    color: #999;
    margin: 0 auto;
    border: 0;
    padding: 10px;
    background: rgba(94, 3, 89, 0.808);
    width: 100%;
    font-weight: 600;
    border-radius: 10px;
    cursor: pointer;
    margin-bottom: 2em;
}
.reply-container{
    padding: 10px;
    display: flex;
    flex-direction: row;
    background: #ccc;
    align-items:center;
    justify-content:center;
}
.reply-container .content{
    flex: 1;
    align-items:center;
    justify-content:center;
}
</style>