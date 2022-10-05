<script>
// This starter template is using Vue 3 <script setup> SFCs
// Check out https://vuejs.org/api/sfc-script-setup.html#script-setup
import HelloWorld from './components/HelloWorld.vue'
import kuzzle from "./services/kuzzle"


export default{
  name: "App",
  data: () => {
    return{
      message: "",
      messages: [],
      roomID: "",
      username: "",
      validate: false,
    }
  },  
  methods: {
    async valid() {
      // Etablish the connection
      await kuzzle.connect();
      // Check if 'chat' index exists
      if (! await kuzzle.index.exists("chat")) {
        // If not, create 'chat' index and 'messages' collection
        await kuzzle.index.create("chat");
        await kuzzle.collection.create("chat", "messages");
      }
      await this.fetchMessages(); // Une fois la collection effectuée on recuppere les 100 derniers messages pour les affichers sur le front 
      await this.subscribeMessages(); // On va ensuite s'abonner au fil 
      this.validate = true;
    },
    getDate(timestamp) {
      const date = new Date(timestamp)
      return date.toLocaleDateString().split("GMT")[0]
    },

    async fetchMessages(){
      const results = await kuzzle.document.search(
        "chat", // Dans l'index chat
        "messages", // Dans la collection messages
        {sort: ["_kuzzle_info.createdAt"]}, // On les tris par ordre de création
        {size: 100} // On prend les 100 premiers 
      )
      // Pour tout les resultats dans results on append la liste messages du message (qu'on extrait du json avec this.getMessage)
      results.hits.map(hit => {
        this.messages = [this.getMessage(hit), ...this.messages]
      })
    },

    getMessage(document){
      const message = {
        _id: document._id,
        value: document._source.value,
        createdAt: document._source._kuzzle_info.createdAt,
        username: document._source.username
      }
      return message
    },
    async sendMessage(){
      if(this.message === "") return;

      // On crée un document avec le sdk
      await kuzzle.document.create(
        "chat", // dans l'index chat
        "messages", // et dans la collection message
        {value: this.message,
        username: this.username}
      )
      this.message = "" // on clear le champ message
    },

    async subscribeMessages(){

      // on va stocker l'id de la room dans this.roomID 
      // On va s'abonner à une collection pour recevoir une notification quand un élement qui correspond à notre filtre est ajouté / modifié / supprimé 
      this.roomID = await kuzzle.realtime.subscribe(
        "chat", // Dans l'index chat
        "messages", // Dans la collection messages
        {}, // Aucun filtre

        // fonction callback (quand on reçoit une notif)
        notification => {
          if(notification.type !== "document") return // on cancel si ce n'est pas pas une notif de document
          if(notification.action !== "create") return // On cancel si ce n'est pas du à la création d'un document
          this.messages = [this.getMessage(notification.result),...this.messages] // on append la liste messages du nouveau document message recu depuis la notif 
        }
      )
    }
  }
}




</script>

<template>
  <h1>{{this.roomID}}</h1>
  <div v-if="!validate">
    <input type="text" autofocus v-on:keyup.enter="valid" v-model="username" placeholder="Enter your nickname"/>
    
    <button @click="valid">Valid</button>
  </div>

  <div v-if="validate">
    <div v-for="message in messages" :key="messages._id" :class="`messages ${message.username === username ? 'fromMe' : 'fromOthers'}`">
      <span class="username">{{ message.username }}</span>
    <span>({{ getDate(message.createdAt) }})</span>
    <p>{{ message.value }}</p>
    </div>
    <div class="wrapper">
      <input
        autofocus
        type="text"
        v-model="message"
        v-on:keyup.enter="sendMessage"
        placeholder="Enter your message"
      />
      <button @click="sendMessage">Send</button>
</div>
  </div>

  
  
</template>

<style scoped>
.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
}
.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}

#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
.username {
  font-weight: bold;
}
.messages {
  padding: 10px;
  margin: 1px;
  width: 45vw;
  border-radius: 10px;
  color: black;
}
.fromMe {
  text-align: right;
  float: right;
  margin-left: 49vw;
  background-color: #9ca4f0;
}
.fromOthers {
  text-align: left;
  margin-right: 49vw;
  float: left;
  background-color: rgb(206, 246, 147);
}

.wrapper {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 10px;
}
</style>
