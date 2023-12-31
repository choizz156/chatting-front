<template>
  <h2>chat</h2>
  <div>
    <div class="chat-container" id="chat-page">
      <div class="users-list">
        <div class="users-list-container">
          <h2>Online Users</h2>
          <ul id="connectedUsers">
            <connected-user
              :my-nickname="myNickname"
              :my-user-id="myUserId"
              v-for="(receiver, index) in receivers"
              :key="receiver.userId"
              :index="index"
              :receiver-id="receiver.userId"
              :receiver-nickname="receiver.nickname"
              :length="receivers.length"
              :make-room="true"
              :is-active="active(receiver.userId)"
              :is-nr-msg="nrMsg(receiver.userId)"
              @add-receiver="addMessageEvent"
              @user-clicked="applyActiveEvent"
              @read-msg="readMessageEvent"
            ></connected-user>
          </ul>
        </div>
        <div>
          <div><strong>nickname:</strong> {{ myNickname }}</div>
          <div><strong>email: </strong>{{ myEmail }}</div>
        </div>
        <hr class="my-2" />
        <div>
          <button
            @click="logout"
            type="button"
            class="btn btn-warning btn-outline-primary"
          >
            logout
          </button>
        </div>
      </div>
      <chat-area
        :stomp-client="stompClient"
        :room-id="selectedRoomId"
        :receiver-id="receiverId"
        :receiver-nickname="receiverNickname"
      ></chat-area>
    </div>
  </div>
</template>

<script>
import SockJS from 'sockjs-client';
import { Stomp } from '@stomp/stompjs';
import ConnectedUser from '@/components/chat/Connected-User.vue';
import ChatArea from '@/components/chat/Chat-Area.vue';
import axios from 'axios';

axios.defaults.withCredentials = true;

export default {
  name: 'Chat-Room',
  components: { ChatArea, ConnectedUser },
  data() {
    return {
      receivers: [],
      isActive: false,
      activeUserId: null,
      myUserId: parseInt(this.$route.query.userId),
      myEmail: this.$route.query.email,
      myNickname: this.$route.query.nickname,
      selectedRoomId: null,
      chatArea: null,
      stompClient: null,
      receiverId: null,
      receiverNickname: '',
      sendingUserId: null,
    };
  },
  computed: {},
  methods: {
    async logout() {
      const host = 'https://choizz-chat.r-e.kr';
      await axios.delete(host + '/auth/logout');
      this.stompClient.disconnect;
      this.$router.push('/');
      alert('로그아웃됐습니다.');
    },
    nrMsg(receiverUserId) {
      return this.sendingUserId === receiverUserId;
    },
    active(receiverUserId) {
      return this.activeUserId === receiverUserId;
    },

    readMessageEvent() {
      this.sendingUserId = null;
    },

    applyActiveEvent(receiverId) {
      this.activeUserId = receiverId;
    },

    addMessageEvent(receiverId, receiverNickname, roomId) {
      this.receiverId = receiverId;
      this.receiverNickname = receiverNickname;
      this.selectedRoomId = roomId;
    },

    connectWebSocket() {
      const headers = {
        'user-info': this.myUserId,
      };
      const socket = new SockJS('https://choizz-chat.r-e.kr/chat');
      this.stompClient = Stomp.over(socket);
      this.stompClient.connect(headers, this.onConnected, this.onError);
    },

    receiveErrorMessage() {
      alert('통신 오류가 발생했습니다.');
      this.$router.push('/login');
    },

    receiveConnectedUser(payload) {
      let connectedUsers = JSON.parse(payload.body);
      this.receivers = connectedUsers.filter(
        (user) => user.email !== this.myEmail
      );
    },
    onConnected() {
      this.stompClient.subscribe(
        `/queue/${this.myUserId}/messages`,
        this.receiveMessage
      );
      this.stompClient.subscribe(
        `/topic/${this.myUserId}/error`,
        this.receiveErrorMessage
      );
      this.stompClient.subscribe(`/topic/error`, this.receiveErrorMessage);
      this.stompClient.subscribe(`/topic/public`, this.receiveConnectedUser);
    },

    onError() {
      alert('소켓 연결 실패');
    },

    async receiveMessage(payload) {
      const message = JSON.parse(payload.body);
      if (this.selectedRoomId === message.roomId) {
        this.$store.commit('addMessage', message);
        return;
      }
      this.sendingUserId = message.senderId;
    },
  },

  mounted() {
    this.connectWebSocket();
    const userInfo = {
      email: this.myEmail,
      nickname: this.myNickname,
      userId: this.myUserId,
    };
    this.$store.commit('setUser', userInfo);

    // this.connectUsers();
  },
};
</script>

<style>
body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  margin: 0;
  padding: 0;
  background-color: #f0f0f0;
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100vh;
  flex-direction: column;
}

.chat-container {
  display: flex;
  max-width: 800px;
  min-width: 800px;
  min-height: 600px;
  max-height: 600px;
  margin: 20px;
  border: 1px solid #ccc;
  background-color: cornsilk;
  overflow: hidden;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  border-radius: 8px;
}

.users-list {
  flex: 1;
  border-right: 1px solid #ccc;
  padding: 20px;
  box-sizing: border-box;
  background-color: burlywood;
  color: #fff;
  border-top-left-radius: 8px;
  border-bottom-left-radius: 8px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

.users-list-container {
  height: 100%;
  overflow-y: scroll;
}

.users-list h2 {
  font-size: 1.5rem;
  margin-bottom: 10px;
}

.users-list ul {
  list-style: none;
  padding: 0;
  margin: 0;
}

a.logout {
  color: #fff;
  text-decoration: none;
}
</style>
