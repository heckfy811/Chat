<template>
  <div class="chat-list">
    <h5>Чаты</h5>
    <button @click="showCreateChatModal = true" class="create-chat-btn">Создать новый чат</button>
    <ul>
      <li v-for="chat in chats" :key="chat.id" 
          :class="{ active: chat.id === currentChatId, muted: isChatMuted(chat.id) }"
          @click="selectChat(chat.id)">
        <span class="chat-name">{{ chat.name }}</span>
        <button @click.stop="toggleMute(chat.id)" class="mute-btn">
          {{ isChatMuted(chat.id) ? '&#x1F515;' : '&#x1F514;' }} <!-- 🔕 / 🔔 -->
        </button>
      </li>
    </ul>
    <p v-if="!chats || chats.length === 0">Нет доступных чатов.</p>

    <!-- Create Chat Modal -->
    <div v-if="showCreateChatModal" class="modal-overlay" @click.self="showCreateChatModal = false">
      <div class="modal-content">
        <h3>Создать новый чат</h3>
        <form @submit.prevent="handleCreateChat">
          <div>
            <label for="chatName">Название чата:</label>
            <input type="text" id="chatName" v-model="newChatName" required>
          </div>
          <div>
            <label for="chatUsers">Добавить пользователей (имена пользователей через запятую):</label>
            <input type="text" id="chatUsers" v-model="newChatUsernames" required>
          </div>
          <div v-if="createChatError" class="error-message">{{ createChatError }}</div>
          <button type="submit">Создать чат</button>
          <button type="button" @click="showCreateChatModal = false">Отмена</button>
        </form>
      </div>
    </div>

  </div>
</template>

<script>
export default {
  name: 'ChatList',
  props: {
    chats: {
      type: Array,
      required: true,
    },
    currentChatId: {
      type: String,
      default: null,
    }
  },
  data() {
    return {
      showCreateChatModal: false,
      newChatName: '',
      newChatUsernames: '',
      createChatError: null,
      mutedChatIds: new Set(), // For muted chat IDs
    };
  },
  created() {
    this.loadMutedChats();
  },
  methods: {
    loadMutedChats() {
      const muted = localStorage.getItem('mutedChatIds');
      if (muted) {
        this.mutedChatIds = new Set(JSON.parse(muted));
      }
    },
    saveMutedChats() {
      localStorage.setItem('mutedChatIds', JSON.stringify(Array.from(this.mutedChatIds)));
    },
    isChatMuted(chatId) {
      return this.mutedChatIds.has(chatId);
    },
    toggleMute(chatId) {
      if (this.mutedChatIds.has(chatId)) {
        this.mutedChatIds.delete(chatId);
      } else {
        this.mutedChatIds.add(chatId);
      }
      this.saveMutedChats();
      this.$emit('chatMuteToggled', { chatId, isMuted: this.isChatMuted(chatId) });
      // Force re-render if needed, though Vue should detect Set changes in methods for classes
      // For deep reactivity or explicit updates, consider Vue.set or forceUpdate in Vue 2.
      // In Vue 3, reactivity with Set should work fine when methods modifying it are called.
    },
    selectChat(chatId) {
      this.$emit('selectChat', chatId);
    },
    async handleCreateChat() {
      this.createChatError = null;
      if (!this.newChatName.trim() || !this.newChatUsernames.trim()) {
        this.createChatError = "Требуется название чата и имена пользователей.";
        return;
      }
      try {
        const usernames = this.newChatUsernames.split(',').map(name => name.trim()).filter(name => name);
        if (usernames.length === 0) {
          this.createChatError = "Пожалуйста, введите хотя бы одно имя пользователя для добавления в чат.";
          return;
        }

        // Emit an event to the parent component (e.g., App.vue or a main view)
        // to handle the actual API call.
        this.$emit('createChat', { name: this.newChatName, userUsernames: usernames });

        // Reset form and close modal on success (or handle this in parent after successful API call)
        this.newChatName = '';
        this.newChatUsernames = '';
        this.showCreateChatModal = false;
      } catch (error) {
        console.error('Ошибка создания чата:', error);
        this.createChatError = error.response?.data?.message || 'Не удалось создать чат. Пожалуйста, попробуйте еще раз.';
        // No need to close modal on error, user might want to correct input
      }
    }
  },
};
</script>

<style scoped src="../assets/chat.css"></style>