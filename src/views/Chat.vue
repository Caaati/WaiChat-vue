<template>
  <div class="app-container">
    <div class="contacts-sidebar">
      <div class="sidebar-header">
        <h3>联系人</h3>
      </div>
      <ul class="contacts-list">
        <li class="add-chat-item" @click="showAddContactModal = true">
          <div class="add-chat-icon">+</div>
          <div class="add-chat-text">新增聊天</div>
        </li>
        <li
          v-for="contact in contacts"
          :key="contact.id"
          :class="{ 'active': selectedContactId === contact.id }"
          @click="selectContact(contact)"
        >
          <div class="contact-avatar">
            <span>{{ contact.nickname.charAt(0) }}</span>
          </div>
          <div class="contact-info">
            <div class="contact-name">
              <span class="nickname">{{ contact.nickname }}</span>
              <span class="username">@{{ contact.username }}</span>
            </div>
            <div class="last-message">{{ contact.lastMessage }}</div>
          </div>
          <span
            v-if="unreadCounts[contact.id] && unreadCounts[contact.id] > 0"
            class="unread-badge"
          >
            {{ unreadCounts[contact.id] }}
          </span>
        </li>
      </ul>
    </div>

    <div class="chat-container">

      <transition name="slide-fade">
        <div v-if="notification.show" class="chat-notification" :class="notification.type">
          <span class="notify-icon">{{ notification.type === 'error' ? '⚠️' : 'ℹ️' }}</span>
          <span class="notify-text">{{ notification.message }}</span>
        </div>
      </transition>

      <div class="chat-header">
        <div class="header-left">
          <div class="contact-avatar" v-if="currentContactName">
            <span>{{ currentContactName.charAt(0) }}</span>
          </div>
          <h2>{{ currentContactName || '选择一个联系人开始聊天' }}</h2>
          <button v-if="selectedContactId" @click="handleClearHistory" class="clear-history-btn" title="清空历史记录">🗑️</button>
          <button v-if="selectedContactId" @click="handleRecoverHistory" class="clear-history-btn" title="恢复历史记录">🔄️</button>
          <button
            class="summary-btn"
            @click="handleSummarize"
            :disabled="aiProcessing || !filteredMessages.length"
            title="总结当前聊天记录"
          >
            📋 总结
          </button>
        </div>

        <div class="translation-controls" v-if="selectedContactId">
          <label class="switch-label" title="收到消息将自动翻译为指定语言">
            <input type="checkbox" v-model="autoTranslate">
            <span class="switch-text">自动翻译</span>
          </label>

          <select v-model="targetLang" class="lang-select">
            <option v-for="lang in languages" :key="lang.code" :value="lang.code">
              {{ lang.flag }} {{ lang.name }}
            </option>
          </select>
        </div>

        <div class="current-user-item">
          <div class="contact-avatar">
            <span>{{ nickname ? nickname.charAt(0) : '' }}</span>
          </div>
          <div class="contact-info">
            <div class="contact-name">
              <span class="nickname">{{ nickname || '未登录' }}</span>
            </div>
            <button class="logout-btn" @click="handleLogout">退出</button>
          </div>
        </div>
      </div>

      <div class="chat-messages">
        <div v-if="messages.length === 0 && selectedContactId" class="empty-chat-hint">
          <p>这里是空的，发送一条消息开始对话吧！</p>
        </div>
        <div v-else-if="!selectedContactId" class="empty-chat-hint">
          <p>请在左侧选择一个联系人。</p>
        </div>
        <div class="message-list">
          <div
            v-for="msg in filteredMessages"
            :key="msg.id"
            :class="['message-item', msg.senderId === userId ? 'self-message' : 'other-message']"
          >
            <div class="message-bubble">
              <div class="message-sender-container">
                <div class="message-sender">
                  {{ msg.senderName || (msg.senderId === userId ? '我' : '未知用户') }}
                </div>
                <div class="message-time">
                  {{ formatTime(msg.timestamp) }}
                </div>
              </div>

              <div class="message-content">{{ msg.content }}</div>

              <div v-if="msg.translatedContent" class="translation-content">
                <div class="divider"></div>
                <div class="translation-line">
                  <div>
                    <span class="trans-icon">{{ getFlag(msg.translatedToLang) }}</span> {{ msg.translatedContent }}
                  </div>
                  <button class="clear-trans-btn" @click.stop="clearTranslation(msg)" title="清除翻译">
                    ❌
                  </button>
                </div>
              </div>
              <div v-else-if="msg.isTranslating" class="translating-spinner">
                翻译中...
              </div>

              <button
                v-if="!msg.translatedContent && msg.senderId !== userId && !msg.isTranslating"
                class="manual-trans-btn"
                @click="translateSingleMessage(msg)"
              >
                翻译
              </button>
            </div>
          </div>
        </div>
      </div>

      <div class="chat-input-wrapper">
        <transition name="slide-up">
          <div v-if="aiSuggestion" class="ai-suggestion-box">
            <div class="suggestion-text">
              <strong>{{ aiSuggestionType === 'polish' ? 'AI 润色建议:' : 'AI 智能回复:' }}</strong>
              {{ aiSuggestion }}
            </div>
            <div class="suggestion-actions">
              <button @click="applySuggestion" class="apply-btn">采纳 (Enter)</button>
              <button @click="cancelSuggestion" class="cancel-btn">取消 (Esc)</button>
            </div>
          </div>
        </transition>
        <div class="ai-toolbar" v-if="selectedContactId">
          <button
            class="ai-tool-btn"
            @click="handleSmartReply"
            :disabled="aiProcessing || !!aiSuggestion"
            title="根据历史记录生成下一句回复"
          >
            🤖 智能回复
          </button>
          <button
            class="ai-tool-btn"
            @click="handleAiPolish('business')"
            :disabled="!message.trim() || aiProcessing || !!aiSuggestion"
            title="将输入文本调整为正式商务风格"
          >
            ✨ 商务润色
          </button>
          <button
            class="ai-tool-btn"
            @click="handleAiPolish('casual')"
            :disabled="!message.trim() || aiProcessing || !!aiSuggestion"
            title="将输入文本调整为友好休闲风格"
          >
            😎 语气软化
          </button>

          <div class="ai-loading" v-if="aiProcessing">AI 思考中...</div>
        </div>

        <div class="chat-input-area">
          <div class="emoji-container">
            <button
              class="emoji-toggle-btn"
              @click.stop="showEmojiPicker = !showEmojiPicker"
              title="选择表情"
            >
              😊
            </button>

            <transition name="slide-fade-fast">
              <div v-if="showEmojiPicker" class="emoji-picker-wrapper">
                <EmojiPicker
                  :native="true"
                  @select="insertEmoji"
                  :theme="'light'"
                />
              </div>
            </transition>
          </div>
          <input
            type="text"
            v-model="message"
            ref="messageInput" @keyup.enter="!aiSuggestion && sendMessage()"
            placeholder="输入消息..."
            class="message-input"
          />
          <button @click="sendMessage" class="send-button">发送</button>
        </div>
      </div>
      <transition name="modal-fade">
        <div v-if="showSummaryModal" class="chat-summary-modal-overlay">
          <div class="chat-summary-modal">
            <div class="modal-header">
              <h3>聊天摘要</h3>
              <button @click="showSummaryModal = false" class="close-btn">×</button>
            </div>
            <div class="modal-content">
              <div v-if="chatSummary" class="summary-text">{{ chatSummary }}</div>
              <div v-else>正在生成摘要...</div>
            </div>
            <div class="modal-footer">
              <button @click="copySummary" class="copy-btn">复制摘要</button>
            </div>
          </div>
        </div>
      </transition>
    </div>

    <AddContactModal
      :is-visible="showAddContactModal"
      :current-user-id="userId"
      @close="showAddContactModal = false"
      @user-selected="handleUserSelected"
      @search-error="showNotification"
    />
  </div>
</template>

<script>
import axios from "axios";
import AddContactModal from '@/components/chat/AddContactModal.vue';
import {CODES} from "@/constants/codes.js";
import EmojiPicker from 'vue3-emoji-picker';

export default {
  components: {
    AddContactModal,
    EmojiPicker
  },
  data() {
    return {
      messages: [],
      languages: [],
      message: '',
      aiSuggestion: '',
      aiSuggestionType: '',
      ws: null,
      userId: null,
      username: null,
      nickname: null,
      selectedContactId: null,
      currentContactName: '',
      contacts: [],
      unreadCounts: {},
      showAddContactModal: false,

      // --- 状态 ---
      autoTranslate: false, // 是否开启自动翻译
      targetLang: 'zh',     // 默认目标语言
      aiProcessing: false,  // AI 是否正在处理
      showSummaryModal: false, // 控制摘要模态框显示
      chatSummary: '',         // 存储摘要文本
      showEmojiPicker: false,  // 控制 Emoji 面板显示
      notification: {
        show: false,
        message: '',
        type: 'info',
        timer: null
      }
    };
  },
  computed: {
    filteredMessages() {
      if (!this.selectedContactId || !this.userId) return [];
      return this.messages
        .filter(msg =>
          (msg.senderId == this.userId && msg.targetId == this.selectedContactId) ||
          (msg.senderId == this.selectedContactId && msg.targetId == this.userId)
        )
        .sort((a, b) => new Date(a.timestamp) - new Date(b.timestamp));
    },
  },

  methods: {
    getLanguages() {
      axios.get('/api/ai/languages')
        .then(res => {
          if (res.data.code === CODES.SUCCESS) {
            this.languages = res.data.data;
            // 如果列表不为空且当前没有选中语言，默认选中第一个
            if (this.languages.length > 0 && !this.targetLang) {
              this.targetLang = this.languages[0].code;
            }
          }
        })
        .catch(e => console.error("获取语言列表失败", e));
    },
    // 插入 Emoji 到输入框
    insertEmoji(emojiObject) {
      // vue3-emoji-picker 默认返回 {i: '😀', n: 'grinning face', ...}
      const emoji = emojiObject.i;
      const input = this.$refs.messageInput;

      if (input && emoji) {
        const start = input.selectionStart;
        const end = input.selectionEnd;

        // 插入 Emoji 到光标位置
        this.message = this.message.substring(0, start) + emoji + this.message.substring(end);

        // 重新设置光标位置
        this.$nextTick(() => {
          input.focus();
          // 将光标设置到新插入文本的末尾
          input.setSelectionRange(start + emoji.length, start + emoji.length);
        });

        // 插入后保持面板开启，方便用户连续插入
      } else if (emoji) {
        this.message += emoji;
      }
    },
    // 复制摘要到剪贴板
    copySummary() {
      if (this.chatSummary) {
        navigator.clipboard.writeText(this.chatSummary).then(() => {
          this.showNotification('摘要已复制到剪贴板');
        }).catch(err => {
          this.showNotification('复制失败', 'error');
          console.error('无法复制文本: ', err);
        });
      }
    },

    // 【新增】处理聊天摘要功能
    async handleSummarize() {
      if (!this.selectedContactId || this.aiProcessing) return;
      if (!this.filteredMessages.length) {
        this.showNotification('当前聊天记录为空，无法总结', 'warning');
        return;
      }

      this.aiProcessing = true;
      this.chatSummary = '';
      this.showSummaryModal = true; // 立即打开模态框，显示加载状态

      // 准备数据：格式化 userId -> 我/对方
      const chatsForSummarize = this.filteredMessages
        .map(m => ({ userId: m.senderId === this.userId ? '我' : '对方', content: m.content }));

      try {
        // 调用后端新的 summarize 接口
        const response = await axios.post('/api/ai/summarize', chatsForSummarize);

        if (response.data.code === CODES.SUCCESS && response.data.data) {
          this.chatSummary = response.data.data.trim();
        } else {
          this.chatSummary = '摘要生成失败，请稍后再试。';
          this.showNotification(response.data.msg || '摘要生成失败', 'error');
        }
      } catch (e) {
        console.error(e);
        this.chatSummary = '摘要服务连接失败。';
        this.showNotification('AI 服务繁忙，请稍后再试', 'error');
      } finally {
        this.aiProcessing = false;
      }
    },
    handleGlobalKeyup(event) {
      if (this.aiSuggestion) {
        if (event.key === 'Enter') {
          event.preventDefault(); // 阻止默认的 Enter 行为
          this.applySuggestion();
        } else if (event.key === 'Escape') {
          event.preventDefault();
          this.cancelSuggestion();
        }
      }
    },
    clearTranslation(msg) {
      if (msg) {
        msg.translatedContent = null;
        msg.translatedToLang = null;
        this.$forceUpdate();
      }
    },
    // 采纳 AI 润色建议
    applySuggestion() {
      if (!this.aiSuggestion) return;
      this.message = this.aiSuggestion;
      this.aiSuggestion = '';
      this.aiSuggestionType = '';
      this.$nextTick(() => document.querySelector('.message-input')?.focus());
    },

    // 取消 AI 润色建议
    cancelSuggestion() {
      this.aiSuggestion = '';
      this.aiSuggestionType = '';
      this.$nextTick(() => document.querySelector('.message-input')?.focus());
    },
    // 旗帜映射辅助函数
    getFlag(code) {
      return code || '🌐';
    },

    // 智能回复功能
    async handleSmartReply() {
      if (!this.selectedContactId) return;
      // 前置检查
      if (this.aiProcessing) {
        this.showNotification('AI 正在处理上一个请求，请稍候', 'warning');
        return;
      }
      if (this.aiSuggestion) {
        this.showNotification('请先处理当前的 AI 建议 (Enter/Esc)', 'warning');
        return;
      }
      // 准备数据：取最近20条，格式化 userId -> 我/对方
      const chatsForSmartReply = this.filteredMessages
        .slice(-20)
        .map(m => ({ userId: m.senderId === this.userId ? '我' : '对方', content: m.content }));
      if (chatsForSmartReply.length === 0) {
        this.showNotification('没有足够的聊天记录来生成智能回复', 'warning');
        return;
      }
      this.aiProcessing = true;
      this.aiSuggestion = ''; // 使用通用建议
      this.aiSuggestionType = '';
      try {
        const response = await axios.post('/api/ai/smartReply', chatsForSmartReply);
        if (response.data.code === CODES.SUCCESS && response.data.data) {
          this.aiSuggestion = response.data.data.trim();
          this.aiSuggestionType = 'smartReply'; // 标记类型
          this.showNotification('已生成智能回复，请按 Enter 采纳');
        } else {
          this.showNotification(response.data.msg || 'AI智能回复服务返回错误或结果为空', 'error');
        }
      } catch (e) {
        console.error(e);
        this.showNotification('AI 服务繁忙', 'error');
      } finally {
        this.aiProcessing = false;
      }
    },
    handleLogout() {
      if (confirm('确定要退出登录吗？')) {
        if (this.ws) this.ws.close();
        localStorage.clear();
        this.$router.push('/login');
      }
    },
    getContactList() {
      this.contacts = [];
      axios.get('/api/chat/getContactList', {params: {userId: this.userId}})
        .then(res => {
          if (res.data.code === CODES.SUCCESS) {
            const _data = Array.isArray(res.data.data) ? res.data.data : [];
            this.contacts = _data.map(item => ({
              id: item.id,
              username: item.username,
              nickname: item.nickname,
              lastMessage: item.content || '无消息'
            }));
          }
        });
    },
    handleUserSelected(user) {
      const existingContactIndex = this.contacts.findIndex(c => c.id == user.id);
      if (existingContactIndex !== -1) {
        const contact = this.contacts.splice(existingContactIndex, 1)[0];
        this.contacts.unshift(contact);
      } else {
        this.contacts.unshift({
          id: user.id, username: user.username, nickname: user.nickname, lastMessage: '无消息'
        });
      }
      this.selectContact({id: user.id, nickname: user.nickname, username: user.username});
    },
    formatTime(timestamp) {
      if (!timestamp) return '';
      const date = new Date(timestamp);
      return `${date.getHours().toString().padStart(2, '0')}:${date.getMinutes().toString().padStart(2, '0')}`;
    },
    selectContact(contact) {
      this.selectedContactId = contact.id;
      this.currentContactName = contact.nickname;
      this.messages = [];
      if (this.unreadCounts[contact.id]) this.unreadCounts[contact.id] = 0;

      axios.get('/api/chat/history', {
        params: {userId: this.userId, targetId: contact.id}
      }).then(res => {
        if (res.data.code === CODES.SUCCESS) {
          const historyData = Array.isArray(res.data.data) ? res.data.data : [];
          this.messages = historyData.map(msg => {
            const isSelf = msg.userId == this.userId;
            return {
              id: msg.id || Date.now() + Math.random(),
              senderId: isSelf ? this.userId : msg.userId,
              targetId: isSelf ? contact.id : msg.targetId,
              content: msg.content,
              senderName: isSelf ? '我' : contact.nickname,
              timestamp: msg.timestamp || msg.createTime || new Date(),
              translatedContent: null // 历史记录暂时不加载翻译，如需加载需后端配合存储
            };
          });
          this.scrollToBottom();
        }
      });
    },
    async handleClearHistory() {
      if (!this.selectedContactId) return;
      if (!confirm(`确定要清空与 ${this.currentContactName} 的所有聊天记录吗？`)) {
        return;
      }
      try {
        const response = await axios.post('/api/chat/removeHistory', {
            userId: this.userId,
            targetId: this.selectedContactId
        });
        if (response.data.code === CODES.SUCCESS || response.data.code === 200) {
          // 成功后，清空本地消息列表
          this.messages = [];
          // 移除联系人列表中的最后一条消息显示
          const contact = this.contacts.find(c => c.id == this.selectedContactId);
          if (contact) {
            contact.lastMessage = '无消息';
          }

          this.showNotification('聊天记录已清空', 'info');
        } else {
          this.showNotification(response.data.msg || '清空历史记录失败', 'error');
        }
      } catch (error) {
        console.error('清空历史记录请求失败', error);
        this.showNotification('清空历史记录失败，网络或服务错误', 'error');
      }
    },
    async handleRecoverHistory() {
      if (!this.selectedContactId) return;

      // 提示用户这是一个恢复操作
      if (!confirm(`确定要恢复与 ${this.currentContactName} 之间已逻辑删除的聊天记录吗？`)) {
        return;
      }

      try {
        // 构造请求体所需的 JSON 对象
        const payload = {
          userId: this.userId,
          targetId: this.selectedContactId
        };

        // 发送 POST 请求，将 payload 作为请求体
        const response = await axios.post('/api/chat/recoverHistory', payload);

        if (response.data.code === CODES.SUCCESS || response.data.code === 200) {
          // 1. 成功后，调用 selectContact 方法来刷新本地消息列表
          // selectContact 方法会重新拉取 chat/history 接口的数据
          const currentContact = this.contacts.find(c => c.id == this.selectedContactId);
          if (currentContact) {
            // 传入当前联系人对象，触发消息历史的重新加载
            this.selectContact(currentContact);
          }

          this.showNotification('聊天记录已恢复', 'info');
        } else {
          this.showNotification(response.data.msg || '恢复历史记录失败', 'error');
        }
      } catch (error) {
        console.error('恢复历史记录请求失败', error);
        this.showNotification('恢复历史记录失败，网络或服务错误', 'error');
      }
    },
    // AI 润色功能
    async handleAiPolish(style) {
      if (!this.message.trim()) return;
      if (this.aiProcessing) {
        this.showNotification('AI 正在处理上一个请求，请稍候', 'warning');
        return;
      }
      if (this.aiSuggestion) {
        this.showNotification('请先处理当前的 AI 建议 (Enter/Esc)', 'warning');
        return;
      }
      this.aiProcessing = true;
      this.aiSuggestion = ''; // 使用通用建议
      this.aiSuggestionType = '';
      try {
        const response = await axios.post('/api/ai/polish', {
          text: this.message,
          style: style
        });
        if (response.data.code === CODES.SUCCESS && response.data.data) {
          this.aiSuggestion = response.data.data.trim(); // 使用通用建议
          this.aiSuggestionType = 'polish'; // 标记类型
          this.showNotification(`已完成${style === 'business' ? '商务' : '语气'}润色，请按 Enter 采纳`);
        } else {
          this.showNotification('AI润色服务返回错误或结果为空', 'error');
        }
      } catch (error) {
        this.showNotification('AI服务暂时繁忙', 'error');
      } finally {
        this.aiProcessing = false;
      }
    },

    // --- 调用翻译接口 ---
    async translateSingleMessage(msg) {
      // 防止重复点击
      if (msg.translatedContent || msg.isTranslating) return;
      msg.isTranslating = true;

      try {
        const response = await axios.post('/api/ai/translate', {
          text: msg.content,
          target: this.targetLang
        });

        if (response.data.code === CODES.SUCCESS) {
          msg.translatedContent = response.data.data.translated;
          msg.translatedToLang = this.targetLang;
        } else {
          console.warn('翻译接口返回异常', response.data);
          this.showNotification('翻译失败', 'error');
        }
      } catch (error) {
        console.error("翻译失败", error);
        this.showNotification('翻译服务不可用', 'error');
      } finally {
        msg.isTranslating = false;
        this.$forceUpdate();
      }
    },

    async sendMessage() {
      if (!this.message.trim() || !this.selectedContactId) return;

      const newMessage = {
        id: Date.now(),
        senderId: this.userId,
        senderName: '我',
        targetId: this.selectedContactId,
        targetName: this.currentContactName,
        content: this.message,
        status: 'sending',
        translatedContent: null
      };

      this.messages.push(newMessage);
      this.scrollToBottom();
      const messageContent = this.message;
      this.message = '';

      try {
        const response = await fetch('/api/chat/send', {
          method: 'POST',
          headers: {'Content-Type': 'application/json'},
          body: JSON.stringify({
            userId: this.userId,
            targetId: this.selectedContactId,
            content: messageContent,
          }),
        });
        const data = await response.json();

        if (data.code === CODES.SUCCESS) {
          newMessage.status = 'sent';
          const contactIndex = this.contacts.findIndex(c => c.id == this.selectedContactId);
          if (contactIndex !== -1) {
            this.contacts[contactIndex].lastMessage = messageContent;
            this.contacts.unshift(this.contacts.splice(contactIndex, 1)[0]);
          }
        } else {
          newMessage.status = 'offline';
          this.showNotification(data.msg || '对方不在线', 'error');
        }
      } catch (error) {
        newMessage.status = 'error';
        this.showNotification('发送失败', 'error');
      }
    },

    showNotification(message, type = 'info') {
      // 设置内容
      this.notification.message = message;
      this.notification.type = type;
      this.notification.show = true;
      // 清除上一次的定时器（防抖）
      if (this.notification.timer) {
        clearTimeout(this.notification.timer);
      }
      // 3秒后自动关闭
      this.notification.timer = setTimeout(() => {
        this.notification.show = false;
      }, 3000);
    },

    scrollToBottom() {
      this.$nextTick(() => {
        const messagesEl = this.$el.querySelector('.chat-messages');
        if (messagesEl) messagesEl.scrollTop = messagesEl.scrollHeight;
      });
    }
  },
  mounted() {
    this.userId = localStorage.getItem('userId');
    this.username = localStorage.getItem('username');
    this.nickname = localStorage.getItem('nickname');
    if (!this.username) {
      this.$router.push('/login');
      return;
    }
    this.getContactList();
    this.getLanguages();
    document.addEventListener('keyup', this.handleGlobalKeyup);
    if (this.userId) {
      this.ws = new WebSocket(`ws://localhost:8080/ws/${this.userId}`);
      this.ws.onmessage = (event) => {
        try {
          const data = JSON.parse(event.data);
          const senderId = data.userId || data.senderId;

          const message = {
            id: data.id || Date.now() + Math.random(),
            senderId: senderId,
            targetId: data.targetId,
            content: data.content,
            senderName: this.contacts.find(c => c.id === senderId)?.nickname || '未知用户',
            timestamp: data.createTime || new Date(),
            translatedContent: null,
            isTranslating: false
          };

          if (this.selectedContactId != senderId) {
            this.unreadCounts[senderId] = (this.unreadCounts[senderId] || 0) + 1;
            this.showNotification(`收到来自 "${message.senderName}" 的新消息`);
          } else {
            this.messages.push(message);

            // --- 新增：自动翻译逻辑 ---
            if (this.autoTranslate) {
              this.translateSingleMessage(message);
            }

            this.scrollToBottom();
          }

          const contactIndex = this.contacts.findIndex(c => c.id == senderId);
          if (contactIndex !== -1) {
            this.contacts[contactIndex].lastMessage = message.content;
            this.contacts.unshift(this.contacts.splice(contactIndex, 1)[0]);
          }
        } catch (e) {
          console.warn('WS error', e);
        }
      };
      this.ws.onclose = (event) => {
        console.log('WebSocket 连接已关闭:', event);
        this.showNotification('与服务器连接已断开。', 'error');
        this.$router.push('/login');
      };

      this.ws.onerror = (error) => {
        console.error('WebSocket 发生错误:', error);
        this.showNotification('WebSocket 发生错误。', 'error');
      };
    }
  },
  beforeUnmount() {
    if (this.ws) this.ws.close();
    document.removeEventListener('keyup', this.handleGlobalKeyup);
  }
};
</script>

<style scoped>
/* 保持原有的 .app-container, .contacts-sidebar 等不变，只展示新增和修改的部分 */

.app-container {
  display: flex;
  height: 97vh;
  background-color: #f0f2f5;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

/* Chat Container 需要 relative 定位作为通知的参照物 */
.chat-container {
  position: relative; /* 关键 */
  flex: 1;
  display: flex;
  flex-direction: column;
  height: 100%;
  background-color: #f7f8fa;
}

/* --- 【新通知样式】 --- */
.chat-notification {
  position: absolute;
  top: 70px; /* 位于头部下方 */
  left: 50%;
  transform: translateX(-50%);
  z-index: 1000;

  /* 外观 */
  background-color: #ffffff;
  padding: 10px 20px;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  display: flex;
  align-items: center;
  gap: 8px;
  min-width: 200px;
  justify-content: center;
  border: 1px solid #eee;
}

.chat-notification.error {
  border-left: 4px solid #ff4d4f;
  color: #d32f2f;
}

.chat-notification.info {
  border-left: 4px solid #42b983;
  color: #333;
}

.notify-icon { font-size: 16px; }
.notify-text { font-size: 14px; font-weight: 500; }

/* Vue Transition 动画 */
.slide-fade-enter-active, .slide-fade-leave-active {
  transition: all 0.3s ease;
}
.slide-fade-enter, .slide-fade-leave-to {
  transform: translate(-50%, -20px); /* 向上滑出 */
  opacity: 0;
}

/* --- 侧边栏和列表样式 (保持原样) --- */
.add-chat-item { padding: 12px 16px; display: flex; align-items: center; cursor: pointer; border-bottom: 1px solid #f0f0f0; color: #42b983; font-weight: 500; }
.add-chat-item:hover { background-color: #f0fdf4; }
.add-chat-icon { width: 40px; height: 40px; border-radius: 50%; background-color: #ecfdf5; color: #42b983; display: flex; align-items: center; justify-content: center; font-size: 20px; margin-right: 12px; flex-shrink: 0; }
.add-chat-text { font-size: 15px; }
.contacts-sidebar { width: 260px; background-color: #ffffff; border-right: 1px solid #e9e9eb; display: flex; flex-direction: column; }
.sidebar-header { padding: 16px; border-bottom: 1px solid #e9e9eb; }
.sidebar-header h3 { margin: 0; font-size: 18px; color: #333; }
.contacts-list { list-style: none; padding: 0; margin: 0; overflow-y: auto; flex: 1; }
.contacts-list li { padding: 12px 16px; display: flex; align-items: center; cursor: pointer; border-bottom: 1px solid #f0f0f0; gap: 12px; }
.contacts-list li:hover { background-color: #f5f5f5; }
.contacts-list li.active { background-color: #e8f0fe; border-left: 3px solid #42b983; }
.contact-avatar { width: 40px; height: 40px; border-radius: 50%; background-color: #42b983; color: white; display: flex; align-items: center; justify-content: center; font-weight: bold; flex-shrink: 0; }
.contact-info { display: flex; flex-direction: column; justify-content: center; flex: 1; min-width: 0; }
.unread-badge { display: inline-flex; align-items: center; justify-content: center; width: 18px; height: 18px; background-color: #ff4d4f; color: white; font-size: 12px; border-radius: 50%; margin-left: auto; }
.contact-name { display: flex; align-items: center; gap: 8px; margin-bottom: 4px; }
.nickname { font-size: 15px; font-weight: 600; color: #333; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.username { font-size: 12px; color: #999; }
.last-message { font-size: 12px; color: #666; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }

/* --- 头部样式 --- */
.chat-header {
  padding: 10px 16px;
  border-bottom: 1px solid #e9e9eb;
  background-color: #ffffff;
  display: flex;
  align-items: center;
  justify-content: space-between;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);
  z-index: 10;
}
.header-left { display: flex; align-items: center; gap: 12px; }
.translation-controls { display: flex; align-items: center; gap: 12px; background-color: #f0f2f5; padding: 6px 12px; border-radius: 20px; }
.switch-label { display: flex; align-items: center; cursor: pointer; font-size: 13px; color: #555; gap: 6px; }
.lang-select { border: 1px solid #ddd; border-radius: 4px; padding: 2px 6px; font-size: 12px; outline: none; background: white; }

/* --- 消息区域 --- */
.chat-messages { flex: 1; padding: 20px; overflow-y: auto; background-color: #f7f8fa; }
.empty-chat-hint { text-align: center; color: #999; padding-top: 30%; font-size: 14px; }
.message-list { display: flex; flex-direction: column; gap: 12px; }
.message-item { display: flex; max-width: 80%; }
.self-message { align-self: flex-end; }
.other-message { align-self: flex-start; }
.message-bubble { padding: 10px 14px; border-radius: 18px; position: relative; box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1); max-width: 100%; word-break: break-word; }
.self-message .message-bubble { background-color: #42b983; color: white; border-top-right-radius: 4px; }
.other-message .message-bubble { background-color: #ffffff; color: #333; border: 1px solid #e0e0e0; border-top-left-radius: 4px; }
.message-sender-container { display: flex; justify-content: space-between; align-items: center; margin-bottom: 4px; }
.message-sender { font-size: 12px; opacity: 0.8; font-weight: 500; }
.message-time { font-size: 11px; margin-left: 8px; opacity: 0.7; color: #999; }
.self-message .message-time { color: white; }
.translation-content { margin-top: 8px; font-size: 14px; }
.self-message .translation-content { color: #e6fffa; }
.other-message .translation-content { color: #4a5568; }
.translation-line {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 10px;
}
.clear-trans-btn {
  background: none;
  border: none;
  font-size: 10px;
  color: #999;
  cursor: pointer;
  padding: 0;
  flex-shrink: 0;
  line-height: 1;
  margin-top: 2px;
}
.clear-trans-btn:hover {
  color: #ff4d4f;
}
.divider { height: 1px; background-color: rgba(0,0,0,0.1); margin: 6px 0; }
.self-message .divider { background-color: rgba(255,255,255,0.3); }
.trans-icon { font-size: 12px; margin-right: 4px; }
.translating-spinner { font-size: 12px; margin-top: 4px; opacity: 0.7; font-style: italic; }
.manual-trans-btn { display: block; margin-top: 4px; font-size: 11px; color: #42b983; background: none; border: 1px solid #42b983; border-radius: 10px; padding: 1px 6px; cursor: pointer; }

/* --- 底部输入区 --- */
.chat-input-wrapper { background-color: #ffffff; border-top: 1px solid #e9e9eb; display: flex; flex-direction: column; }
.ai-toolbar { display: flex; gap: 8px; padding: 8px 16px 0 16px; align-items: center; }
.ai-tool-btn { background-color: #f0f9eb; color: #42b983; border: 1px solid #e1f3d8; border-radius: 12px; padding: 4px 10px; font-size: 12px; cursor: pointer; transition: all 0.2s; }
.ai-tool-btn:hover:not(:disabled) { background-color: #42b983; color: white; }
.ai-tool-btn:disabled { opacity: 0.5; cursor: not-allowed; background-color: #f5f5f5; border-color: #ddd; color: #999; }
.ai-loading { font-size: 12px; color: #999; font-style: italic; margin-left: auto; }
.chat-input-area { display: flex; padding: 12px 16px; gap: 10px; align-items: center; }
.message-input { flex: 1; padding: 12px 16px; border: 1px solid #dcdfe6; border-radius: 24px; font-size: 14px; outline: none; }
.message-input:focus { border-color: #42b983; }
.send-button { padding: 12px 24px; background-color: #42b983; color: white; border: none; border-radius: 24px; font-size: 14px; font-weight: 500; cursor: pointer; }
.send-button:hover { background-color: #36a47e; }
.current-user-item { display: flex; align-items: center; padding: 8px 12px; cursor: default; }
.logout-btn { background: none; border: none; font-size: 12px; color: #666; cursor: pointer; padding: 4px 8px; border-radius: 4px; margin-left: 8px; }
.logout-btn:hover { background-color: #f5f5f5; color: #ff4d4f; }

.ai-suggestion-box {
  padding: 10px 16px;
  background-color: #fffbe6; /* 浅黄色背景，突出提示 */
  border-top: 1px solid #fae6b0;
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 14px;
  color: #664d03;
}

.suggestion-text {
  flex: 1;
  margin-right: 20px;
  word-break: break-word;
}

.suggestion-actions {
  display: flex;
  gap: 8px;
  flex-shrink: 0;
}

.apply-btn, .cancel-btn {
  padding: 6px 12px;
  border-radius: 16px;
  font-size: 12px;
  cursor: pointer;
  font-weight: 500;
  transition: all 0.2s;
}

.apply-btn {
  background-color: #42b983;
  color: white;
  border: 1px solid #42b983;
}

.apply-btn:hover {
  background-color: #36a47e;
}

.cancel-btn {
  background-color: #ffffff;
  color: #666;
  border: 1px solid #ccc;
}

.cancel-btn:hover {
  background-color: #f0f0f0;
}

/* 建议区域的动画 */
.slide-up-enter-active, .slide-up-leave-active {
  transition: all 0.2s ease-out;
}
.slide-up-enter, .slide-up-leave-to {
  transform: translateY(100%);
  opacity: 0;
}
/* --- 聊天摘要按钮样式 --- */
.summary-btn {
  background-color: #f5f5f5;
  color: #606266;
  border: 1px solid #dcdfe6;
  border-radius: 4px;
  padding: 6px 12px;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.2s;
  margin-left: 10px; /* 与其他元素保持间距 */
}
.summary-btn:hover:not(:disabled) {
  background-color: #ebebeb;
}
.summary-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}


/* --- 摘要模态框样式 --- */
.chat-summary-modal-overlay {
  position: absolute; /* 相对于 chat-container */
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000; /* 确保在最上层 */
}

.chat-summary-modal {
  background-color: white;
  border-radius: 8px;
  width: 90%;
  max-width: 800px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  overflow: hidden;
  display: flex;
  flex-direction: column;
  max-height: 80vh;
}

.modal-header {
  padding: 15px 20px;
  border-bottom: 1px solid #ebeef5;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.modal-header h3 {
  margin: 0;
  font-size: 18px;
  font-weight: 500;
}

.close-btn {
  background: none;
  border: none;
  font-size: 24px;
  color: #909399;
  cursor: pointer;
  line-height: 1;
}

.modal-content {
  padding: 20px;
  flex-grow: 1;
  overflow-y: auto;
  white-space: pre-wrap; /* 保留LLM输出的分段和换行 */
  font-size: 15px;
  line-height: 1.6;
  color: #303133;
}

.summary-text {
  /* 确保总结文本样式良好 */
}

.modal-footer {
  padding: 15px 20px;
  border-top: 1px solid #ebeef5;
  text-align: right;
}

.copy-btn {
  background-color: #42b983;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 8px 15px;
  cursor: pointer;
  transition: background-color 0.2s;
}

.copy-btn:hover {
  background-color: #36a47e;
}

/* 模态框动画 */
.modal-fade-enter-active, .modal-fade-leave-active {
  transition: opacity 0.3s ease;
}
.modal-fade-enter, .modal-fade-leave-to {
  opacity: 0;
}
.emoji-container {
  position: relative;
  /* 使按钮在输入区居中对齐 */
  align-self: center;
  flex-shrink: 0;
  margin-right: -5px; /* 调整与输入框的间距 */
}

.emoji-toggle-btn {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  padding: 0 10px;
  transition: transform 0.2s;
  line-height: 1;
  color: #606266;
}

.emoji-toggle-btn:hover {
  transform: scale(1.1);
  color: #42b983;
}

/* 专门用于容纳 EmojiPicker 组件的容器 */
.emoji-picker-wrapper {
  position: absolute;
  bottom: 100%; /* 弹出在输入框上方 */
  left: -10px; /* 稍微向左偏移，避免被遮挡 */
  margin-bottom: 10px;
  z-index: 20; /* 确保在最上层 */
}

/* --- 快速淡入淡出动画 --- */
.slide-fade-fast-enter-active, .slide-fade-fast-leave-active {
  transition: all 0.2s ease;
}
.slide-fade-fast-enter, .slide-fade-fast-leave-to {
  transform: translateY(10px);
  opacity: 0;
}
</style>
