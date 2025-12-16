<template>
  <div class="login-container">
    <div class="login-box">
      <h2>WaiChat登录</h2>
      <form @submit.prevent="handleLogin">
        <div class="form-group">
          <label>用户名</label>
          <input type="text" v-model="username" required>
        </div>
        <div class="form-group">
          <label>密码</label>
          <input type="password" v-model="password" required>
        </div>
        <button type="submit" class="login-btn">登录</button>
        <button type="button" class="register-btn" @click="toRegister">注册</button>
      </form>
    </div>
  </div>
</template>

<script>
// ... (Script 保持不变)
import axios from 'axios';
import { useRouter } from 'vue-router';
import {CODES} from "@/constants/codes.js";

export default {
  data() {
    return {
      username: '',
      password: ''
    };
  },
  methods: {
    async handleLogin() {
      try {
        const formData = new FormData();
        formData.append('username', this.username);
        formData.append('password', this.password);

        const res = await axios.post('/api/login', formData);

        if (res.data.code === CODES.SUCCESS) {
          if (res.data.data && res.data.data.id) {
            localStorage.setItem('userId', res.data.data.id);
            localStorage.setItem('username', res.data.data.username);
            localStorage.setItem('nickname', res.data.data.nickname);
          }
          this.$router.push('/chat');
        } else {
          alert(res.data.msg || '登录失败');
        }
      } catch (err) {
        console.error(err);
        alert('网络错误或服务器异常');
      }
    },
    toRegister() {
      this.$router.push('/register');
    }
  }
};
</script>

<style scoped>
.login-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background-color: #f5f5f5;
  padding: 0 1rem; /* 防止在极小屏幕紧贴边缘 */
}

.login-box {
  background: white;
  padding: 2rem;
  border-radius: 8px;
  box-shadow: 0 0 10px rgba(0,0,0,0.1);
  /* 修改宽度为响应式 */
  width: 100%;
  max-width: 400px;
}

.form-group {
  margin-bottom: 1rem;
}

input {
  width: 100%;
  padding: 0.5rem;
  margin-top: 0.3rem;
  box-sizing: border-box; /* 确保padding不撑大宽度 */
}

.login-btn, .register-btn {
  width: 100%;
  padding: 0.5rem;
  margin-top: 1rem;
  background: #42b983;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.register-btn {
  background: #2196f3;
}
</style>
