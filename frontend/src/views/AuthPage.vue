<template>
  <div class="auth-page-wrapper">
    <div class="auth-container">
      <div class="logo-section">
        <img src="/logo.png" alt="Alexicon Logo" />
      </div>
      <div class="form-section">
        <h1 class="welcome-message">Добро пожаловать в Chatting!</h1>
        <div class="auth-form">
          <div class="tabs">
            <button :class="{ active: isLogin }" @click="switchToLogin">Войти</button>
            <button :class="{ active: !isLogin }" @click="switchToRegister">Регистрация</button>
          </div>

          <div v-if="errorMessage" class="error-message">{{ errorMessage }}</div>
          <div v-if="successMessage" class="success-message">{{ successMessage }}</div>

          <form @submit.prevent="handleSubmit">
            <div v-if="!isLogin" class="form-group">
              <label for="username">Имя пользователя</label>
              <input type="text" id="username" v-model="form.username" required />
            </div>
            <div class="form-group">
              <label for="email">Электронная почта</label>
              <input type="email" id="email" v-model="form.email" required />
            </div>
            <div class="form-group">
              <label for="password">Пароль</label>
              <input type="password" id="password" v-model="form.password" required />
            </div>
            <div v-if="!isLogin" class="form-group">
              <label for="confirmPassword">Подтвердите пароль</label>
              <input type="password" id="confirmPassword" v-model="form.confirmPassword" required />
            </div>
            <button type="submit" :disabled="isLoading">{{ isLoading ? 'Обработка...' : (isLogin ? 'Войти' : 'Регистрация') }}</button>
          </form>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'AuthPage',
  data() {
    return {
      isLogin: true,
      form: {
        username: '',
        email: '',
        password: '',
        confirmPassword: '',
      },
      errorMessage: '',
      successMessage: '',
      isLoading: false,
    };
  },
  methods: {
    clearMessages() {
      this.errorMessage = '';
      this.successMessage = '';
    },
    resetForm() {
      this.form.username = '';
      this.form.email = '';
      this.form.password = '';
      this.form.confirmPassword = '';
    },
    switchToLogin() {
      this.isLogin = true;
      this.clearMessages();
      this.resetForm();
    },
    switchToRegister() {
      this.isLogin = false;
      this.clearMessages();
      this.resetForm();
    },
    async handleSubmit() {
      this.clearMessages();
      this.isLoading = true;

      if (this.isLogin) {
        // Handle login
        try {
          const response = await fetch('http://localhost:3000/login', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ email: this.form.email, password: this.form.password }),
          });
          const data = await response.json();
          if (!response.ok) {
            throw new Error(data.message || 'Ошибка входа');
          }
          localStorage.setItem('authToken', data.token);
          this.successMessage = 'Вход выполнен успешно! Перенаправление...';
          // Optionally reset form: this.resetForm();
          this.$router.push('/chat');
        } catch (error) {
          this.errorMessage = error.message || 'Во время входа произошла ошибка.';
        }
      } else {
        // Handle registration
        if (this.form.password !== this.form.confirmPassword) {
          this.errorMessage = "Пароли не совпадают!";
          this.isLoading = false;
          return;
        }
        try {
          const response = await fetch('http://localhost:3000/register', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
              username: this.form.username,
              email: this.form.email,
              password: this.form.password,
            }),
          });
          const data = await response.json();
          if (!response.ok) {
            throw new Error(data.message || 'Ошибка регистрации');
          }
          this.successMessage = 'Регистрация прошла успешно! Пожалуйста, войдите.';
          this.resetForm();
          this.isLogin = true; // Switch to login tab
        } catch (error) {
          this.errorMessage = error.message || 'Во время регистрации произошла ошибка.';
        }
      }
      this.isLoading = false;
    },
  },
};
</script>

<style scoped>
.auth-page-wrapper {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
  background-color: #F0F2F5; /* Light gray background for the whole page */
  color: #333333; /* Default dark text color */
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  width: 100%;
  box-sizing: border-box;
}

.error-message {
  color: red;
  background-color: #ffebee;
  padding: 10px;
  border-radius: 4px;
  margin-bottom: 15px;
  text-align: center;
}

.success-message {
  color: green;
  background-color: #e8f5e9;
  padding: 10px;
  border-radius: 4px;
  margin-bottom: 15px;
  text-align: center;
}
</style>

<style scoped src="../assets/auth.css"></style> 