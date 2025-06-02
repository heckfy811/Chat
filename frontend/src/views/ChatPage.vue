<template>
  <div class="chat-page">
    <div class="chat-list-container">
      <div class="account-section">
        <button @click="toggleAccountDropdown" class="account-btn" ref="accountBtn">
          <span class="account-icon">👤</span>
          <span class="account-text">{{ currentUserDetails && currentUserDetails.username ? currentUserDetails.username : 'Аккаунт' }}</span>
        </button>
        <div v-if="showAccountDropdown" class="account-dropdown" v-click-outside="{ handler: closeAccountDropdown, exclude: [$refs.accountBtn] }">
          <div v-if="currentUserDetails">
            <p v-if="currentUserDetails.username"><strong>Имя пользователя:</strong> {{ currentUserDetails.username }}</p>
            <p v-if="currentUserDetails.email"><strong>Электронная почта:</strong> {{ currentUserDetails.email }}</p>
            <p><strong>ID:</strong> {{ currentUserDetails.id }}</p>
          </div>
          <button @click="logout" class="logout-btn">Выйти</button>
        </div>
      </div>
      <ChatList 
        :chats="chatList" 
        :currentChatId="selectedChatId" 
        @selectChat="handleSelectChat" 
        @createChat="handleCreateChat"
        @chatMuteToggled="handleChatMuteToggled" />
    </div>
    <div class="chat-view-container">
      <ChatView 
        :messages="currentChatMessages" 
        :currentChatId="selectedChatId" 
        :currentUserId="currentUserId" 
        :chatParticipants="currentChatParticipants" 
        :isLoadingMessages="isLoadingMessages"
        @sendMessage="handleSendMessage" 
        @toggleReaction="handleToggleReaction" />
    </div>
  </div>
</template>

<script>
import io from 'socket.io-client';
import ChatList from '@/components/ChatList.vue';
import ChatView from '@/components/ChatView.vue';
// import Vue from 'vue'; // Not needed for Vue 3 with composition API style reactivity updates

export default {
  name: 'ChatPage',
  directives: {
    clickOutside: {
      mounted(el, binding) {
        el.__ClickOutsideHandler__ = event => {
          let excluded = false;
          if (binding.value.exclude) {
            for (const excludedEl of binding.value.exclude) {
              if (excludedEl && (excludedEl === event.target || excludedEl.contains(event.target))) {
                excluded = true;
                break;
              }
            }
          }
          if (!excluded && !(el === event.target || el.contains(event.target))) {
            binding.value.handler(event);
          }
        };
        document.body.addEventListener('click', el.__ClickOutsideHandler__);
        // Store the handler on the element so it can be removed correctly
        el.__ClickOutsideBinding__ = binding.value;
      },
      beforeUnmount(el) {
        document.body.removeEventListener('click', el.__ClickOutsideHandler__);
      },
      // Handle binding updates if necessary, though for this case it might not be needed
      // updated(el, binding) {
      //   el.__ClickOutsideBinding__ = binding.value;
      // }
    }
  },
  components: {
    ChatList,
    ChatView,
  },
  data() {
    return {
      socket: null,
      allMessagesByChat: {}, // Store messages per chat: { chatId1: [msg1, msg2], chatId2: [...] }
      chatList: [], 
      selectedChatId: null, 
      currentUserId: null, 
      currentUserDetails: null, // To store more details like username
      currentChatParticipants: [], 
      isLoadingMessages: false,
      mutedChatIds: new Set(), // To store IDs of muted chats
      showAccountDropdown: false, // For account dropdown visibility
    };
  },
  computed: {
    currentChatMessages() {
      if (!this.selectedChatId || !this.allMessagesByChat[this.selectedChatId]) {
        return [];
      }
      // Return a sorted copy of messages by timestamp
      return [...this.allMessagesByChat[this.selectedChatId]].sort((a, b) => new Date(a.timestamp) - new Date(b.timestamp));
    }
  },
  methods: {
    decodeToken() {
      const token = localStorage.getItem('authToken');
      if (token) {
        try {
          const base64Url = token.split('.')[1];
          const base64 = base64Url.replace(/-/g, '+').replace(/_/g, '/');
          const jsonPayload = decodeURIComponent(atob(base64).split('').map(function(c) {
              return '%' + ('00' + c.charCodeAt(0).toString(16)).slice(-2);
          }).join(''));
          const decodedToken = JSON.parse(jsonPayload);
          this.currentUserId = decodedToken.id;
          this.currentUserDetails = { 
            id: decodedToken.id, 
            username: decodedToken.username, 
            email: decodedToken.email // Assuming email is in the token
          };
        } catch (e) {
          console.error('Ошибка декодирования токена:', e);
          this.currentUserId = null;
          this.currentUserDetails = null;
        }
      } else {
        console.warn('Токен авторизации не найден. ID пользователя не может быть установлен.');
        this.currentUserDetails = null;
      }
    },

    toggleAccountDropdown() {
      this.showAccountDropdown = !this.showAccountDropdown;
    },

    closeAccountDropdown() { // Method to close the dropdown
      this.showAccountDropdown = false;
    },

    logout() {
      console.log('Выход из системы...');
      localStorage.removeItem('authToken');
      localStorage.removeItem('mutedChatIds'); // Clear muted chats preference on logout
      this.currentUserId = null;
      this.currentUserDetails = null;
      this.chatList = [];
      this.allMessagesByChat = {};
      this.selectedChatId = null;
      this.mutedChatIds = new Set();
      if (this.socket) {
        this.socket.disconnect();
      }
      this.$router.push('/login');
    },

    playNotificationSound() {
      // Check if the current selected chat is muted
      if (this.selectedChatId && this.mutedChatIds.has(this.selectedChatId)) {
        console.log(`Чат ${this.selectedChatId} отключен. Звук уведомления подавлен.`);
        return; 
      }
      const audio = new Audio('/notification.mp3'); // Assumes notification.mp3 is in /public
      audio.play().catch(error => console.warn("Ошибка воспроизведения звука уведомления:", error));
    },

    loadMutedChatsFromStorage() { // Renamed to avoid conflict with ChatList's method if ever mixed
      const muted = localStorage.getItem('mutedChatIds');
      if (muted) {
        this.mutedChatIds = new Set(JSON.parse(muted));
        console.log('ChatPage: Загружены отключенные чаты из localStorage:', Array.from(this.mutedChatIds));
      }
    },

    handleSelectChat(chatId) {
      if (this.selectedChatId === chatId && this.allMessagesByChat[chatId]) return; // Avoid re-joining/re-loading if already selected and messages loaded

      if (this.selectedChatId && this.socket && this.socket.connected) {
        this.socket.emit('leave_chat', this.selectedChatId);
        console.log(`Покинул чат: ${this.selectedChatId}`);
      }
      
      this.selectedChatId = chatId;
      this.isLoadingMessages = true;
      // Initialize message array for the chat if it doesn't exist
      if (!this.allMessagesByChat[chatId]) {
        this.allMessagesByChat = {
          ...this.allMessagesByChat,
          [chatId]: []
        };
      }

      const selectedChat = this.chatList.find(chat => chat.id === chatId);
      if (selectedChat) {
        if (selectedChat.participants && selectedChat.participantUsernames && selectedChat.participants.length === selectedChat.participantUsernames.length) {
            this.currentChatParticipants = selectedChat.participants.map((id, index) => ({
                id: id,
                username: selectedChat.participantUsernames[index]
            }));
        } else {
            this.currentChatParticipants = selectedChat.participants?.map(id => ({id, username: `Пользователь ${id.substring(0,4)}`})) || [];
            console.warn("Информация об именах участников может быть неполной для ChatView для чата:", chatId);
        }
      } else {
        this.currentChatParticipants = [];
        console.warn("Выбранный чат не найден в chatList:", chatId);
      }

      if (this.socket && this.socket.connected) {
        this.socket.emit('join_chat', chatId);
        console.log(`Запрошено присоединение к чату: ${chatId}`);
      } else {
        console.error("Сокет не подключен, невозможно присоединиться к чату. Попытка присоединения при подключении.");
        // isLoading will be reset by load_messages or if socket fails to connect and join
      }
    },

    handleCreateChat(chatDetails) {
      console.log('Попытка создать чат:', chatDetails);
      const token = localStorage.getItem('authToken');
      fetch('http://localhost:3000/api/chats', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${token}`,
        },
        body: JSON.stringify(chatDetails),
      })
      .then(response => {
        if (!response.ok) {
          return response.json().then(err => { throw new Error(err.message || 'Не удалось создать чат'); });
        }
        return response.json();
      })
      .then(newChat => {
        console.log('Чат создан:', newChat);
        this.chatList.push(newChat);
        this.handleSelectChat(newChat.id); // Select and join the new chat
      })
      .catch(error => {
        console.error('Ошибка создания чата:', error.message);
        alert(`Ошибка создания чата: ${error.message}`);
      });
    },

    handleSendMessage(messageData) {
      if (!this.currentUserId) {
        console.error('Невозможно отправить сообщение, currentUserId не установлен.');
        alert('Ошибка: Пользователь не идентифицирован. Пожалуйста, войдите снова.');
        return;
      }

      // Updated validation: chat_id must exist.
      // If it's a text message, text must be non-empty.
      // If it's a file message (image/video), url must exist.
      if (!messageData.chatId) {
        console.error('Невозможно отправить сообщение, chatId отсутствует.', messageData);
        return;
      }

      if (messageData.type === 'text') {
        if (!messageData.text || !messageData.text.trim()) {
          console.warn('Попытка отправить пустое текстовое сообщение.');
          return;
        }
      } else if (messageData.type === 'image' || messageData.type === 'video') {
        if (!messageData.url) {
          console.error('Невозможно отправить файловое сообщение, URL отсутствует.', messageData);
          return;
        }
      } else {
        // Should not happen if ChatView emits correctly
        console.error('Неизвестный тип сообщения или тип отсутствует:', messageData);
        return;
      }
      
      if (!this.socket || !this.socket.connected) {
        console.error('Сокет не подключен. Невозможно отправить сообщение.');
        alert('Нет подключения к серверу чата. Пожалуйста, проверьте ваше соединение.');
        return;
      }

      const messageToSend = {
        ...messageData,
        senderId: this.currentUserId, 
        timestamp: new Date().toISOString(),
      };
      this.socket.emit('send_message', messageToSend);
    },

    fetchChats() {
        const token = localStorage.getItem('authToken');
        if (!token) {
            console.warn("Токен авторизации не найден, невозможно загрузить чаты.");
            // Potentially redirect to login or show an error message
            return Promise.reject("Токен авторизации не найден");
        }
        this.isLoadingMessages = true; // Indicate loading for initial chat/message load
        return fetch('http://localhost:3000/api/chats', {
            headers: {
                'Authorization': `Bearer ${token}`,
            },
        })
        .then(response => {
            if (!response.ok) {
                throw new Error('Не удалось загрузить чаты');
            }
            return response.json();
        })
        .then(chats => {
            this.chatList = chats;
            if (chats.length > 0 && !this.selectedChatId) {
                this.handleSelectChat(chats[0].id); // Auto-select and join first chat
            } else {
                this.isLoadingMessages = false; // No chats or one already selected, stop loading indicator
            }
        })
        .catch(error => {
            console.error('Ошибка загрузки чатов:', error);
            alert('Ошибка загрузки ваших чатов. Пожалуйста, попробуйте позже.');
            this.isLoadingMessages = false;
        });
    },

    handleToggleReaction(reactionData) {
      console.log('ChatPage: получено handleToggleReaction:', reactionData);
      if (!this.currentUserId || !this.selectedChatId) {
        console.error('ChatPage: Пользователь или чат не идентифицирован для реакции. currentUserId:', this.currentUserId, 'selectedChatId:', this.selectedChatId);
        return;
      }
      if (!this.socket || !this.socket.connected) {
        console.error('ChatPage: Сокет не подключен. Невозможно отправить реакцию.');
        alert('Нет подключения к серверу чата. Пожалуйста, проверьте ваше соединение, чтобы отреагировать.');
        return;
      }
      const payload = {
        ...reactionData, // { messageId, reactionEmoji }
        userId: this.currentUserId,
        chatId: this.selectedChatId 
      };
      console.log('ChatPage: Отправка toggle_reaction на сервер с полезной нагрузкой:', payload);
      this.socket.emit('toggle_reaction', payload);
    },

    handleChatMuteToggled({ chatId, isMuted }) {
      if (isMuted) {
        this.mutedChatIds.add(chatId);
      } else {
        this.mutedChatIds.delete(chatId);
      }
      // Optionally, re-save to ChatPage's localStorage if you want redundancy,
      // but ChatList already handles persistence. Here, we just update the reactive state.
      console.log(`ChatPage: Статус отключения звука для чата ${chatId} обновлен на ${isMuted}. Отключенные чаты:`, Array.from(this.mutedChatIds));
    },

    setupSocketListeners() {
        if (!this.socket) return;

        this.socket.on('connect', () => {
          console.log('Подключено к WebSocket серверу:', this.socket.id, 'для пользователя:', this.currentUserId);
          // If a chat was selected before socket connected (e.g. from fetchChats), join it.
          if (this.selectedChatId) {
            this.socket.emit('join_chat', this.selectedChatId);
            console.log(`Повторный запрос на присоединение к чату после подключения: ${this.selectedChatId}`);
          } else if (this.chatList.length > 0) {
            // If no chat was selected but chats are loaded, select and join the first one.
            this.handleSelectChat(this.chatList[0].id);
          }
        });

        this.socket.on('load_messages', (data) => {
          console.log(`Получены load_messages для чата ${data.chatId}:`, data.messages);
          const sortedMessages = [...data.messages].sort((a, b) => new Date(a.timestamp) - new Date(b.timestamp));
          this.allMessagesByChat = {
            ...this.allMessagesByChat,
            [data.chatId]: sortedMessages
          };
          this.isLoadingMessages = false;
        });

        this.socket.on('receive_message', (newMessage) => {
          console.log('Получено новое сообщение:', newMessage);
          const { chatId, senderId } = newMessage; // Destructure senderId
          const existingMessages = this.allMessagesByChat[chatId] ? [...this.allMessagesByChat[chatId]] : [];
          
          // Avoid duplicates if message somehow already exists (e.g. sent by self and echoed)
          if (!existingMessages.find(msg => msg.id === newMessage.id)) {
            const updatedMessages = [...existingMessages, newMessage].sort((a,b) => new Date(a.timestamp) - new Date(b.timestamp));
            this.allMessagesByChat = {
              ...this.allMessagesByChat,
              [chatId]: updatedMessages
            };

            // Play sound notification if message is from another user and chat is selected
            if (senderId !== this.currentUserId && chatId === this.selectedChatId) {
              this.playNotificationSound();
            }
          }
        });

        // New listener for reaction updates
        this.socket.on('message_reaction_updated', (data) => {
          const { messageId, reactions } = data;
          if (!this.selectedChatId || !this.allMessagesByChat[this.selectedChatId]) {
            console.warn('Получено обновление реакции для чата, который не активен или не имеет локально загруженных сообщений.', data);
            return;
          }
          
          const messageIndex = this.allMessagesByChat[this.selectedChatId].findIndex(m => m.id === messageId);
          if (messageIndex > -1) {
            // Create a new object for the message to ensure reactivity for nested `reactions`
            const updatedMessage = {
              ...this.allMessagesByChat[this.selectedChatId][messageIndex],
              reactions: reactions || {} // Ensure reactions is an object
            };
            
            // Create a new array for the chat messages to ensure reactivity of the list
            const updatedChatMessages = [...this.allMessagesByChat[this.selectedChatId]];
            updatedChatMessages[messageIndex] = updatedMessage;

            this.allMessagesByChat = {
              ...this.allMessagesByChat,
              [this.selectedChatId]: updatedChatMessages
            };
            console.log('Реакция обновлена для сообщения:', messageId, 'Новые реакции:', reactions);
          } else {
            console.warn('Получено обновление реакции для сообщения, не найденного локально:', messageId);
          }
        });

        this.socket.on('disconnect', () => {
          console.log('Отключено от WebSocket сервера');
        });

        this.socket.on('connect_error', (err) => {
            console.error("Ошибка подключения к сокету:", err.message);
            alert("Не удалось подключиться к серверу чата. Пожалуйста, проверьте ваше соединение или попробуйте позже.");
            this.isLoadingMessages = false;
        });
    }
  },
  async created() {
    this.decodeToken();
    this.loadMutedChatsFromStorage(); // Load muted status
    if (this.currentUserId) {
        await this.fetchChats(); // Wait for chats to be fetched before setting up socket
        this.socket = io('http://localhost:3000', {
            query: { userId: this.currentUserId } 
        });
        this.setupSocketListeners();
    } else {
        console.error("ID пользователя недоступен, функциональность чата отключена.");
        alert("Вы не вошли в систему. Пожалуйста, войдите, чтобы использовать чат.");
        // Consider redirecting to login page: this.$router.push('/login');
    }
  },
  beforeUnmount() {
    if (this.socket) {
      if (this.selectedChatId) {
        this.socket.emit('leave_chat', this.selectedChatId);
      }
      this.socket.disconnect();
    }
  },
};
</script>

<style scoped src="../assets/chat.css"></style>