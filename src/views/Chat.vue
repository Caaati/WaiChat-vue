<template>
  <div class="app-container" :class="{ 'mobile-mode': isMobile }">
    <transition name="slide-side">
      <div class="contacts-sidebar" v-show="sidebarVisible" id="sidebar">
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
            :class="{ active: selectedContactId === contact.id }"
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
    </transition>

    <div class="chat-container">
      <transition name="slide-fade">
        <div v-if="notification.show" class="chat-notification" :class="notification.type">
          <span class="notify-icon">{{ notification.type === 'error' ? '⚠️' : 'ℹ️' }}</span>
          <span class="notify-text">{{ notification.message }}</span>
        </div>
      </transition>

      <div class="chat-header">
        <div class="header-left">
          <button class="toggle-sidebar-btn" @click="toggleSidebar" title="切换侧边栏/返回列表">
            <span v-if="isMobile">⬅️</span> <span v-else>{{ sidebarVisible ? '◀' : '▶' }}</span>
          </button>

          <div class="contact-avatar" v-if="currentContactName">
            <span>{{ currentContactName.charAt(0) }}</span>
          </div>
          <h2 class="chat-title">{{ currentContactName || '未选择联系人' }}</h2>

          <div class="header-actions">
            <button
              v-if="selectedContactId"
              @click="handleClearHistory"
              class="icon-btn"
              title="清空历史记录"
            >
              🗑️
            </button>
            <button
              v-if="selectedContactId"
              @click="handleRecoverHistory"
              class="icon-btn"
              title="恢复历史记录"
            >
              🔄️
            </button>
            <button
              class="summary-btn mobile-hide-text"
              @click="handleSummarize"
              :disabled="aiProcessing || !filteredMessages.length"
              title="总结当前聊天记录"
            >
              📋 <span class="btn-text">总结</span>
            </button>
            <button
              class="analysis-btn mobile-hide-text"
              @click="handleAnalysis"
              :disabled="aiProcessing || !filteredMessages.length"
              title="查看聊天数据分析看板"
            >
              📊 <span class="btn-text">分析</span>
            </button>
          </div>
        </div>

        <div class="header-right">
          <div class="translation-controls" v-if="selectedContactId && !isMobileSimple">
            <label class="switch-label" title="收到消息将自动翻译为指定语言">
              <input type="checkbox" v-model="autoTranslate" />
              <span class="switch-text">自动翻译</span>
            </label>

            <select v-model="targetLang" class="lang-select">
              <option v-for="lang in languages" :key="lang.code" :value="lang.code">
                {{ lang.flag }} {{ lang.name }}
              </option>
            </select>
          </div>

          <div class="current-user-item">
            <div class="contact-avatar" v-if="!isMobileSimple">
              <span>{{ nickname ? nickname.charAt(0) : '' }}</span>
            </div>
            <div class="contact-info">
              <div class="contact-name" v-if="!isMobileSimple">
                <span class="nickname">{{ nickname || '未登录' }}</span>
              </div>
              <button class="logout-btn" @click="handleLogout">
                {{ isMobileSimple ? '退出' : '退出登录' }}
              </button>
            </div>
          </div>
        </div>
      </div>

      <div class="chat-messages">
        <div v-if="messages.length === 0 && selectedContactId" class="empty-chat-hint">
          <p>这里是空的，发送一条消息开始对话吧！</p>
        </div>
        <div v-else-if="!selectedContactId" class="empty-chat-hint">
          <p v-if="!isMobile">请点击左侧折叠按钮或选择一个联系人。</p>
          <p v-else>点击左上角返回联系人列表</p>
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

              <div
                v-if="msg.type === 'VOICE'"
                class="voice-content"
                @click="playVoice(msg.audioUrl)"
              >
                <span class="voice-icon">
                  {{ isPlaying === msg.audioUrl ? '🔊...' : '🔊' }}
                </span>
                <span class="voice-duration">{{ msg.duration || 0 }}"</span>
              </div>
              <div class="message-content">
                <template v-if="msg.type === 'TEXT' || !msg.type">
                  {{ msg.content }}
                </template>

                <template v-else-if="msg.type === 'VOICE'">
                  <div class="voice-player" @click="playAudio(msg.audioUrl)">
                    <span class="voice-icon"> <i class="wifi-icon"></i> 🎤 </span>
                    <span class="voice-duration">{{ msg.duration }}"</span>
                    <audio :src="msg.audioUrl" ref="audioPlayer"></audio>
                  </div>
                </template>
              </div>

              <div v-if="msg.translatedContent" class="translation-content">
                <div class="divider"></div>
                <div class="translation-line">
                  <div>
                    <span class="trans-icon">{{ getFlag(msg.translatedToLang) }}</span>
                    {{ msg.translatedContent }}
                  </div>
                  <button
                    class="clear-trans-btn"
                    @click.stop="clearTranslation(msg)"
                    title="清除翻译"
                  >
                    ❌
                  </button>
                </div>
              </div>
              <div v-else-if="msg.isTranslating" class="translating-spinner">翻译中...</div>

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

      <div class="chat-input-wrapper" v-if="selectedContactId">
        <transition name="slide-up">
          <div v-if="aiSuggestion" class="ai-suggestion-box">
            <div class="suggestion-text">
              <strong>{{ aiSuggestionType === 'polish' ? 'AI 润色建议:' : 'AI 智能回复:' }}</strong>
              {{ aiSuggestion }}
            </div>
            <div class="suggestion-actions">
              <button @click="applySuggestion" class="apply-btn">采纳</button>
              <button @click="cancelSuggestion" class="cancel-btn">取消</button>
            </div>
          </div>
        </transition>
        <div class="ai-toolbar">
          <button
            class="mic-trigger-btn"
            @touchstart.prevent="handleTouchStart"
            @touchmove.prevent="handleTouchMove"
            @touchend.prevent="handleTouchEnd"
          >
            🎤
          </button>
          <button
            class="ai-tool-btn"
            @click="handleSmartReply"
            :disabled="aiProcessing || !!aiSuggestion"
          >
            🤖 <span class="btn-text">智能回复</span>
          </button>
          <button
            class="ai-tool-btn"
            @click="handleAiPolish('business')"
            :disabled="!message.trim() || aiProcessing || !!aiSuggestion"
          >
            ✨ <span class="btn-text">商务润色</span>
          </button>
          <button
            class="ai-tool-btn"
            @click="handleAiPolish('casual')"
            :disabled="!message.trim() || aiProcessing || !!aiSuggestion"
          >
            😎 <span class="btn-text">语气软化</span>
          </button>

          <div class="ai-loading" v-if="aiProcessing">Thinking...</div>
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
                <EmojiPicker :native="true" @select="insertEmoji" :theme="'light'" />
              </div>
            </transition>
          </div>
          <input
            type="text"
            v-model="message"
            ref="messageInput"
            @keyup.enter="!aiSuggestion && sendMessage()"
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
      <transition name="modal-fade">
        <div v-if="showAnalysisModal" class="analysis-modal-overlay">
          <div class="analysis-modal">
            <div class="modal-header">
              <h3>🤖 AI 聊天数据看板</h3>
              <button @click="showAnalysisModal = false" class="close-btn">×</button>
            </div>

            <div class="modal-content analysis-content">
              <div v-if="analysisLoading" class="analysis-loading">
                <div class="loading-spinner"></div>
                <p>AI 正在分析你们的聊天数据...</p>
              </div>

              <div v-else-if="analysisData" class="dashboard-grid">
                <div class="dashboard-card summary-card">
                  <div class="card-title">✨ AI 关系透视</div>
                  <div class="ai-comment">{{ analysisData.summary }}</div>
                </div>

                <div class="dashboard-card">
                  <div class="chart-container" ref="keywordChart"></div>
                </div>

                <div class="dashboard-card">
                  <div class="chart-container" ref="sentimentChart"></div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </transition>
    </div>
    <transition name="fade">
      <div v-if="isRecording" class="recording-overlay">
        <div class="recording-card">
          <div v-if="recordStatus === 'cancel'" class="status-icon cancel">🗑️</div>
          <div v-else-if="recordStatus === 'transcribe'" class="status-icon transcribe">📝</div>
          <div v-else class="status-icon pulse">🎤</div>
          <p :class="['status-hint', { warning: recordStatus !== 'normal' }]">{{ statusHint }}</p>
        </div>
        <div class="record-zones">
          <div class="zone left" :class="{ active: recordStatus === 'cancel' }">取消发送</div>
          <div class="zone right" :class="{ active: recordStatus === 'transcribe' }">转文字</div>
        </div>
      </div>
    </transition>
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
import axios from 'axios'
import AddContactModal from '@/components/chat/AddContactModal.vue'
import { CODES } from '@/constants/codes.js'
import EmojiPicker from 'vue3-emoji-picker'
import * as echarts from 'echarts'

export default {
  components: {
    AddContactModal,
    EmojiPicker,
  },
  data() {
    return {
      isRecording: false,
      recordStatus: 'normal', // normal, cancel, transcribe
      startX: 0,
      startY: 0,
      mediaRecorder: null,
      audioChunks: [],
      recordStartTime: 0,
      isPlaying: null, // 当前播放的音频地址

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
      autoTranslate: false,
      targetLang: 'zh',
      aiProcessing: false,
      showSummaryModal: false,
      chatSummary: '',
      showEmojiPicker: false,

      // ***** 看板相关状态 *****
      showAnalysisModal: false,
      analysisData: null,
      analysisLoading: false,
      chartInstanceKeywords: null,
      chartInstanceSentiment: null,

      notification: {
        show: false,
        message: '',
        type: 'info',
        timer: null,
      },

      // --- 响应式适配新增 ---
      sidebarVisible: true, // 控制侧边栏是否可见
      isMobile: false, // 是否为移动端
    }
  },
  computed: {
    statusHint() {
      if (this.recordStatus === 'cancel') return '松开手指，取消发送'
      if (this.recordStatus === 'transcribe') return '松开手指，转文字'
      return '手指上划，取消或转文字'
    },
    filteredMessages() {
      if (!this.selectedContactId || !this.userId) return []
      return this.messages
        .filter(
          (msg) =>
            (msg.senderId == this.userId && msg.targetId == this.selectedContactId) ||
            (msg.senderId == this.selectedContactId && msg.targetId == this.userId),
        )
        .sort((a, b) => new Date(a.timestamp) - new Date(b.timestamp))
    },
    // 极简模式（屏幕非常窄时）
    isMobileSimple() {
      return this.isMobile && window.innerWidth < 400
    },
  },

  methods: {
    playAudio(url) {
      if (!url) return;
      const audio = new Audio(url);
      audio.play().catch(err => {
        console.error("播放失败:", err);
        this.$message.error("播放失败，请检查文件链接是否有效");
      });
    },
    async handleStrangerMessage(senderId, content) {
      try {
        // 复用 AddContactModal 中的接口逻辑
        const response = await axios.get('/api/user/search', {
          params: { key: String(senderId) }, // 确保转为字符串
        })

        if (response.data.code === CODES.SUCCESS) {
          const users = response.data.data
          // 精确匹配 ID (防止搜索结果返回多个相似 ID 的情况)
          const user = users.find((u) => u.id == senderId)

          if (user) {
            // 1. 构造新的联系人对象
            const newContact = {
              id: user.id,
              username: user.username,
              nickname: user.nickname,
              lastMessage: content,
              // 如果有头像字段也可以加在这里
            }

            // 2. 添加到左侧列表顶部
            this.contacts.unshift(newContact)

            // 3. 【视觉优化】更新刚才收到的那条显示为 "未知用户" 或 "ID" 的消息的名字
            // 遍历当前消息列表，把该用户发的消息名字修正过来
            this.messages.forEach((msg) => {
              if (msg.senderId == senderId) {
                msg.senderName = user.nickname
              }
            })

            // 4. 如果不需要强制更新视图，Vue 的响应式通常会自动处理
            // 但如果消息列表没有变化，可以尝试 this.$forceUpdate() (一般不需要)
          }
        }
      } catch (error) {
        console.error('获取陌生人信息失败', error)
      }
    },
    checkScreenSize() {
      this.isMobile = window.innerWidth <= 768
      // 如果切换到桌面端，默认展开侧边栏；如果切换到移动端，根据是否有选人决定
      if (!this.isMobile) {
        this.sidebarVisible = true
      } else {
        // 移动端：如果有选中联系人，侧边栏隐藏（显示聊天）；否则显示侧边栏
        this.sidebarVisible = !this.selectedContactId
      }
    },
    toggleSidebar() {
      if (this.isMobile) {
        // 移动端：此按钮充当“返回”键
        this.selectedContactId = null
        this.sidebarVisible = true
      } else {
        // 桌面端：此按钮充当折叠/展开键
        this.sidebarVisible = !this.sidebarVisible
      }
    },

    // 原有方法修改：选择联系人时
    selectContact(contact) {
      this.selectedContactId = contact.id
      this.currentContactName = contact.nickname
      this.messages = []
      if (this.unreadCounts[contact.id]) this.unreadCounts[contact.id] = 0

      // 移动端：选择后隐藏侧边栏进入聊天
      if (this.isMobile) {
        this.sidebarVisible = false
      }

      axios
        .get('/api/chat/history', {
          params: { userId: this.userId, targetId: contact.id },
        })
        .then((res) => {
          if (res.data.code === CODES.SUCCESS) {
            const historyData = Array.isArray(res.data.data) ? res.data.data : []
            this.messages = historyData.map((msg) => {
              const isSelf = msg.userId == this.userId;
              return {
                id: msg.id,
                senderId: isSelf ? this.userId : msg.userId,
                targetId: isSelf ? contact.id : msg.targetId,
                content: msg.content,
                // --- 核心修复：添加以下三个字段 ---
                type: msg.type,
                audioUrl: msg.audioUrl,
                duration: msg.duration,
                // ------------------------------
                senderName: isSelf ? '我' : contact.nickname,
                timestamp: msg.createTime, // 后端返回的是 createTime
                translatedContent: null,
              };
            })
            this.scrollToBottom()
          }
        })
    },

    // ... 其他原有方法保持不变 (getLanguages, handleAnalysis, initCharts, insertEmoji, etc.) ...
    getLanguages() {
      axios
        .get('/api/ai/languages')
        .then((res) => {
          if (res.data.code === CODES.SUCCESS) {
            this.languages = res.data.data
            if (this.languages.length > 0 && !this.targetLang) {
              this.targetLang = this.languages[0].code
            }
          }
        })
        .catch((e) => console.error('获取语言列表失败', e))
    },
    async handleAnalysis() {
      if (!this.selectedContactId || this.analysisLoading) return
      if (this.filteredMessages.length < 5) {
        this.showNotification('聊天记录太少，无法进行有效分析', 'warning')
        return
      }
      this.showAnalysisModal = true
      this.analysisLoading = true
      this.analysisData = null
      const chatsForAnalysis = this.filteredMessages.map((m) => ({
        userId: m.senderId === this.userId ? '我' : '对方',
        content: m.content,
      }))
      try {
        const response = await axios.post('/api/ai/analysis', chatsForAnalysis)
        if (response.data.code === CODES.SUCCESS && response.data.data) {
          const result = response.data.data
          if (!result.keywords || !result.sentiment) {
            this.showNotification('AI 数据不完整，无法显示图表', 'error')
            this.showAnalysisModal = false
            return
          }
          this.analysisData = result
          this.$nextTick(() => {
            setTimeout(() => {
              this.initCharts()
            }, 50)
          })
        } else {
          this.showNotification(response.data.msg || '分析失败，请重试', 'error')
          this.showAnalysisModal = false
        }
      } catch (e) {
        console.error(e)
        this.showNotification('AI 服务繁忙', 'error')
        this.showAnalysisModal = false
      } finally {
        this.analysisLoading = false
      }
    },
    initCharts() {
      if (!this.analysisData) return
      const keywordChartDom = this.$refs.keywordChart
      if (keywordChartDom && keywordChartDom.offsetWidth > 0 && keywordChartDom.offsetHeight > 0) {
        if (this.chartInstanceKeywords) {
          this.chartInstanceKeywords.dispose()
          this.chartInstanceKeywords = null
        }
        this.chartInstanceKeywords = echarts.init(keywordChartDom)
        const keywords = this.analysisData.keywords || []
        if (keywords.length === 0) {
          this.chartInstanceKeywords.setOption({
            title: { text: '💬 暂无关键词数据', left: 'center' },
          })
          this.chartInstanceKeywords.setOption({
            title: { text: '💬 暂无关键词数据', left: 'center' },
          })
        } else {
          this.chartInstanceKeywords.setOption({
            title: { text: '💬 高频热词 Top 8', left: 'center', textStyle: { fontSize: 14 } },
            tooltip: {},
            grid: { top: 40, bottom: 20, left: 10, right: 10, containLabel: true },
            xAxis: { type: 'value', show: false },
            yAxis: {
              type: 'category',
              data: keywords.map((k) => k.name).reverse(),
              axisLine: { show: false },
              axisTick: { show: false },
            },
            series: [
              {
                type: 'bar',
                data: keywords.map((k) => k.value).reverse(),
                itemStyle: {
                  color: new echarts.graphic.LinearGradient(0, 0, 1, 0, [
                    { offset: 0, color: '#83bff6' },
                    { offset: 1, color: '#188df0' },
                  ]),
                  borderRadius: [0, 10, 10, 0],
                },
                label: { show: true, position: 'right' },
              },
            ],
          })
        }
      }
      const sentimentChartDom = this.$refs.sentimentChart
      if (
        sentimentChartDom &&
        sentimentChartDom.offsetWidth > 0 &&
        sentimentChartDom.offsetHeight > 0
      ) {
        if (this.chartInstanceSentiment) {
          this.chartInstanceSentiment.dispose()
          this.chartInstanceSentiment = null
        }
        this.chartInstanceSentiment = echarts.init(sentimentChartDom)
        const sentiments = this.analysisData.sentiment || []
        if (sentiments.length === 0) {
          this.chartInstanceSentiment.setOption({
            title: { text: '❤️ 暂无情感波动数据', left: 'center' },
          })
          return
        }
        this.chartInstanceSentiment.setOption({
          title: {
            text: '❤️ 情感波动趋势 (最近10条)',
            left: 'center',
            textStyle: { fontSize: 14 },
          },
          tooltip: { trigger: 'axis' },
          grid: { top: 40, bottom: 20, left: 10, right: 10, containLabel: true },
          xAxis: { type: 'category', data: sentiments.map((_, i) => i + 1) },
          yAxis: {
            type: 'value',
            min: -1,
            max: 1,
            splitNumber: 2,
            axisLabel: {
              formatter: function (value) {
                if (value === 1) return '😊'
                if (value === 0) return '😐'
                if (value === -1) return '😠'
                return ''
              },
            },
          },
          series: [
            {
              data: sentiments,
              type: 'line',
              smooth: true,
              lineStyle: { color: '#ff7043', width: 3 },
              areaStyle: {
                color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
                  { offset: 0, color: 'rgba(255, 112, 67, 0.5)' },
                  { offset: 1, color: 'rgba(255, 112, 67, 0.1)' },
                ]),
              },
            },
          ],
        })
      }
    },
    insertEmoji(emojiObject) {
      const emoji = emojiObject.i
      const input = this.$refs.messageInput
      if (input && emoji) {
        const start = input.selectionStart
        const end = input.selectionEnd
        this.message = this.message.substring(0, start) + emoji + this.message.substring(end)
        this.$nextTick(() => {
          input.focus()
          input.setSelectionRange(start + emoji.length, start + emoji.length)
        })
      } else if (emoji) {
        this.message += emoji
      }
    },
    copySummary() {
      if (this.chatSummary) {
        navigator.clipboard
          .writeText(this.chatSummary)
          .then(() => {
            this.showNotification('摘要已复制到剪贴板')
          })
          .catch((err) => {
            this.showNotification('复制失败', 'error')
            console.error('无法复制文本: ', err)
          })
      }
    },
    async handleSummarize() {
      if (!this.selectedContactId || this.aiProcessing) return
      if (!this.filteredMessages.length) {
        this.showNotification('当前聊天记录为空，无法总结', 'warning')
        return
      }
      this.aiProcessing = true
      this.chatSummary = ''
      this.showSummaryModal = true
      const chatsForSummarize = this.filteredMessages.map((m) => ({
        userId: m.senderId === this.userId ? '我' : '对方',
        content: m.content,
      }))
      try {
        const response = await axios.post('/api/ai/summarize', chatsForSummarize)
        if (response.data.code === CODES.SUCCESS && response.data.data) {
          this.chatSummary = response.data.data.trim()
        } else {
          this.chatSummary = '摘要生成失败，请稍后再试。'
          this.showNotification(response.data.msg || '摘要生成失败', 'error')
        }
      } catch (e) {
        console.error(e)
        this.chatSummary = '摘要服务连接失败。'
        this.showNotification('AI 服务繁忙，请稍后再试', 'error')
      } finally {
        this.aiProcessing = false
      }
    },
    handleGlobalKeyup(event) {
      if (this.aiSuggestion) {
        if (event.key === 'Enter') {
          event.preventDefault()
          this.applySuggestion()
        } else if (event.key === 'Escape') {
          event.preventDefault()
          this.cancelSuggestion()
        }
      }
    },
    clearTranslation(msg) {
      if (msg) {
        msg.translatedContent = null
        msg.translatedToLang = null
        this.$forceUpdate()
      }
    },
    applySuggestion() {
      if (!this.aiSuggestion) return
      this.message = this.aiSuggestion
      this.aiSuggestion = ''
      this.aiSuggestionType = ''
      this.$nextTick(() => document.querySelector('.message-input')?.focus())
    },
    cancelSuggestion() {
      this.aiSuggestion = ''
      this.aiSuggestionType = ''
      this.$nextTick(() => document.querySelector('.message-input')?.focus())
    },
    getFlag(code) {
      return code || '🌐'
    },
    async handleSmartReply() {
      if (!this.selectedContactId) return
      if (this.aiProcessing) {
        this.showNotification('AI 正在处理上一个请求，请稍候', 'warning')
        return
      }
      if (this.aiSuggestion) {
        this.showNotification('请先处理当前的 AI 建议 (Enter/Esc)', 'warning')
        return
      }
      const originalChats = this.filteredMessages.slice(-20).map((m) => ({
        userId: m.senderId === this.userId ? '我' : '对方',
        content: m.content,
      }))

      // 在数组最后（末尾）添加仅包含双方ID的对象
      const chatsForSmartReply = [
        ...originalChats, // 保留原有所有处理后的消息
        {
          // 末尾追加ID对象
          userId: this.userId,
          targetId: this.selectedContactId,
        },
      ]

      if (chatsForSmartReply.length === 0) {
        this.showNotification('没有足够的聊天记录来生成智能回复', 'warning')
        return
      }
      this.aiProcessing = true
      this.aiSuggestion = ''
      this.aiSuggestionType = ''
      try {
        const response = await axios.post('/api/ai/smartReply', chatsForSmartReply)
        if (response.data.code === CODES.SUCCESS && response.data.data) {
          this.aiSuggestion = response.data.data.trim()
          this.aiSuggestionType = 'smartReply'
          this.showNotification('已生成智能回复，请按 Enter 采纳')
        } else {
          this.showNotification(response.data.msg || 'AI智能回复服务返回错误或结果为空', 'error')
        }
      } catch (e) {
        console.error(e)
        this.showNotification('AI 服务繁忙', 'error')
      } finally {
        this.aiProcessing = false
      }
    },
    handleLogout() {
      if (confirm('确定要退出登录吗？')) {
        if (this.ws) this.ws.close()
        localStorage.clear()
        this.$router.push('/login')
      }
    },
    getContactList() {
      this.contacts = []
      axios.get('/api/chat/getContactList', { params: { userId: this.userId } }).then((res) => {
        if (res.data.code === CODES.SUCCESS) {
          const _data = Array.isArray(res.data.data) ? res.data.data : []
          this.contacts = _data.map((item) => ({
            id: item.id,
            username: item.username,
            nickname: item.nickname,
            lastMessage: item.content || '无消息',
          }))
        }
      })
    },
    handleUserSelected(user) {
      const existingContactIndex = this.contacts.findIndex((c) => c.id == user.id)
      if (existingContactIndex !== -1) {
        const contact = this.contacts.splice(existingContactIndex, 1)[0]
        this.contacts.unshift(contact)
      } else {
        this.contacts.unshift({
          id: user.id,
          username: user.username,
          nickname: user.nickname,
          lastMessage: '无消息',
        })
      }
      this.selectContact({ id: user.id, nickname: user.nickname, username: user.username })
    },
    formatTime(timestamp) {
      if (!timestamp) return ''
      const date = new Date(timestamp)
      return `${date.getHours().toString().padStart(2, '0')}:${date.getMinutes().toString().padStart(2, '0')}`
    },
    async handleClearHistory() {
      if (!this.selectedContactId) return
      if (!confirm(`确定要清空与 ${this.currentContactName} 的所有聊天记录吗？`)) {
        return
      }
      try {
        const response = await axios.post('/api/chat/removeHistory', {
          userId: this.userId,
          targetId: this.selectedContactId,
        })
        if (response.data.code === CODES.SUCCESS || response.data.code === 200) {
          this.messages = []
          const contact = this.contacts.find((c) => c.id == this.selectedContactId)
          if (contact) {
            contact.lastMessage = '无消息'
          }
          this.showNotification('聊天记录已清空', 'info')
        } else {
          this.showNotification(response.data.msg || '清空历史记录失败', 'error')
        }
      } catch (error) {
        console.error('清空历史记录请求失败', error)
        this.showNotification('清空历史记录失败，网络或服务错误', 'error')
      }
    },
    async handleRecoverHistory() {
      if (!this.selectedContactId) return
      if (!confirm(`确定要恢复与 ${this.currentContactName} 之间已逻辑删除的聊天记录吗？`)) {
        return
      }
      try {
        const payload = {
          userId: this.userId,
          targetId: this.selectedContactId,
        }
        const response = await axios.post('/api/chat/recoverHistory', payload)
        if (response.data.code === CODES.SUCCESS || response.data.code === 200) {
          const currentContact = this.contacts.find((c) => c.id == this.selectedContactId)
          if (currentContact) {
            this.selectContact(currentContact)
          }
          this.showNotification('聊天记录已恢复', 'info')
        } else {
          this.showNotification(response.data.msg || '恢复历史记录失败', 'error')
        }
      } catch (error) {
        console.error('恢复历史记录请求失败', error)
        this.showNotification('恢复历史记录失败，网络或服务错误', 'error')
      }
    },
    async handleAiPolish(style) {
      if (!this.message.trim()) return
      if (this.aiProcessing) {
        this.showNotification('AI 正在处理上一个请求，请稍候', 'warning')
        return
      }
      if (this.aiSuggestion) {
        this.showNotification('请先处理当前的 AI 建议 (Enter/Esc)', 'warning')
        return
      }
      this.aiProcessing = true
      this.aiSuggestion = ''
      this.aiSuggestionType = ''
      try {
        const response = await axios.post('/api/ai/polish', {
          text: this.message,
          style: style,
        })
        if (response.data.code === CODES.SUCCESS && response.data.data) {
          this.aiSuggestion = response.data.data.trim()
          this.aiSuggestionType = 'polish'
          this.showNotification(
            `已完成${style === 'business' ? '商务' : '语气'}润色，请按 Enter 采纳`,
          )
        } else {
          this.showNotification('AI润色服务返回错误或结果为空', 'error')
        }
      } catch (error) {
        this.showNotification('AI服务暂时繁忙', 'error')
      } finally {
        this.aiProcessing = false
      }
    },
    async translateSingleMessage(msg) {
      if (msg.translatedContent || msg.isTranslating) return
      msg.isTranslating = true
      try {
        const response = await axios.post('/api/ai/translate', {
          text: msg.content,
          target: this.targetLang,
        })
        if (response.data.code === CODES.SUCCESS) {
          msg.translatedContent = response.data.data.translated
          msg.translatedToLang = this.targetLang
        } else {
          console.warn('翻译接口返回异常', response.data)
          this.showNotification('翻译失败', 'error')
        }
      } catch (error) {
        console.error('翻译失败', error)
        this.showNotification('翻译服务不可用', 'error')
      } finally {
        msg.isTranslating = false
        this.$forceUpdate()
      }
    },
    async sendMessage() {
      if (!this.message.trim() || !this.selectedContactId) return
      const newMessage = {
        id: Date.now(),
        senderId: this.userId,
        senderName: '我',
        targetId: this.selectedContactId,
        targetName: this.currentContactName,
        content: this.message,
        status: 'sending',
        translatedContent: null,
      }
      this.messages.push(newMessage)
      this.scrollToBottom()
      const messageContent = this.message
      this.message = ''
      try {
        const response = await fetch('/api/chat/send', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            userId: this.userId,
            targetId: this.selectedContactId,
            content: messageContent,
          }),
        })
        const data = await response.json()
        if (data.code === CODES.SUCCESS) {
          newMessage.status = 'sent'
          const contactIndex = this.contacts.findIndex((c) => c.id == this.selectedContactId)
          if (contactIndex !== -1) {
            this.contacts[contactIndex].lastMessage = messageContent
            this.contacts.unshift(this.contacts.splice(contactIndex, 1)[0])
          }
        } else {
          newMessage.status = 'offline'
          this.showNotification(data.msg || '对方不在线', 'error')
        }
      } catch (error) {
        newMessage.status = 'error'
        this.showNotification('发送失败', 'error')
      }
    },
    // --- 录音手势处理 ---
    async handleTouchStart(e) {
      const touch = e.touches[0]
      this.startX = touch.clientX
      this.startY = touch.clientY
      this.recordStatus = 'normal'

      try {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true })
        this.mediaRecorder = new MediaRecorder(stream)
        this.audioChunks = []
        this.mediaRecorder.ondataavailable = (e) => this.audioChunks.push(e.data)
        this.mediaRecorder.start()
        this.isRecording = true
        this.recordStartTime = Date.now()
        if (navigator.vibrate) navigator.vibrate(20)
      } catch (err) {
        this.showNotification('无法访问麦克风', 'error')
      }
    },
    handleTouchMove(e) {
      if (!this.isRecording) return
      const touch = e.touches[0]
      const offsetX = touch.clientX - this.startX
      const offsetY = touch.clientY - this.startY

      if (offsetY < -80) {
        // 向上滑动超过 80px
        if (offsetX < -40) this.recordStatus = 'cancel'
        else if (offsetX > 40) this.recordStatus = 'transcribe'
        else this.recordStatus = 'normal'
      } else {
        this.recordStatus = 'normal'
      }
    },
    async handleTouchEnd() {
      if (!this.isRecording) return
      this.isRecording = false
      this.mediaRecorder.stop()
      const duration = Math.round((Date.now() - this.recordStartTime) / 1000)

      this.mediaRecorder.onstop = async () => {
        const audioBlob = new Blob(this.audioChunks, { type: 'audio/webm' })
        if (this.recordStatus === 'cancel') {
          this.showNotification('已取消')
        } else if (this.recordStatus === 'transcribe') {
          this.processAudioSTT(audioBlob)
        } else {
          if (duration < 1) {
            this.showNotification('说话时间太短', 'warning')
            return
          }
          this.uploadAndSendVoice(audioBlob, duration)
        }
      }
    },
    // --- 语音上传与发送 ---
    async uploadAndSendVoice(blob, duration) {
      const formData = new FormData()
      formData.append('file', blob, 'voice.webm')
      try {
        const res = await axios.post('/api/upload', formData) // 需后端配合
        const audioUrl = res.data.data
        const voiceMsg = {
          userId: this.userId,
          targetId: this.selectedContactId,
          content: '[语音消息]',
          type: 'VOICE',
          audioUrl: audioUrl,
          duration: duration,
        }
        console.log('准备通过 WebSocket 发送数据:', voiceMsg)
        this.ws.send(JSON.stringify(voiceMsg))
        this.messages.push(voiceMsg)
        this.scrollToBottom()
      } catch (e) {
        this.showNotification('语音发送失败', 'error')
      }
    },
    async processAudioSTT(blob) {
      const formData = new FormData()
      formData.append('file', blob, 'stt.wav')
      this.aiProcessing = true
      try {
        const res = await axios.post('/api/ai/audio/stt', formData)
        if (res.data.code === CODES.SUCCESS) {
          this.message = res.data.data
          this.showNotification('已转写为文字')
        }
      } catch (e) {
        this.showNotification('识别失败', 'error')
      } finally {
        this.aiProcessing = false
      }
    },
    playVoice(url) {
      if (this.isPlaying === url) return
      const audio = new Audio(url)
      this.isPlaying = url
      audio.play()
      audio.onended = () => {
        this.isPlaying = null
      }
    },
    showNotification(message, type = 'info') {
      this.notification.message = message
      this.notification.type = type
      this.notification.show = true
      if (this.notification.timer) {
        clearTimeout(this.notification.timer)
      }
      this.notification.timer = setTimeout(() => {
        this.notification.show = false
      }, 3000)
    },
    scrollToBottom() {
      this.$nextTick(() => {
        const messagesEl = this.$el.querySelector('.chat-messages')
        if (messagesEl) messagesEl.scrollTop = messagesEl.scrollHeight
      })
    },
  },
  mounted() {
    this.userId = localStorage.getItem('userId')
    this.username = localStorage.getItem('username')
    this.nickname = localStorage.getItem('nickname')
    if (!this.username) {
      this.$router.push('/login')
      return
    }
    // 初始化检测屏幕大小
    this.checkScreenSize()
    window.addEventListener('resize', this.checkScreenSize)
    this.getContactList()
    this.getLanguages()
    document.addEventListener('keyup', this.handleGlobalKeyup)
    if (this.userId) {
      this.ws = new WebSocket(`/ws/${this.userId}`)
      this.ws.onmessage = (event) => {
        try {
          const data = JSON.parse(event.data)
          const senderId = data.userId || data.senderId
          // 查找现有联系人
          let contactIndex = this.contacts.findIndex((c) => c.id == senderId)
          let senderName = '未知用户'
          if (contactIndex !== -1) {
            // --- 情况 A: 是熟人 ---
            senderName = this.contacts[contactIndex].nickname
            // 更新左侧列表预览
            this.contacts[contactIndex].lastMessage = data.content
            // 移到顶部
            const existingContact = this.contacts.splice(contactIndex, 1)[0]
            this.contacts.unshift(existingContact)
          } else if (senderId != this.userId) {
            // --- 情况 B: 是陌生人 ---
            senderName = `新朋友` // 暂时显示 ID，等待接口返回昵称
            this.handleStrangerMessage(senderId, data.content)
          }
          // 构建消息对象
          const message = {
            id: data.id || Date.now() + Math.random(),
            senderId: senderId,
            targetId: data.targetId,
            content: data.content,
            type: data.type || 'TEXT', // 接收来自后端的 type 字段
            audioUrl: data.audioUrl, // 接收语音地址
            duration: data.duration, // 接收时长
            senderName: senderName,
            timestamp: data.createTime || new Date(),
            translatedContent: null,
            isTranslating: false,
          }
          if (this.selectedContactId != senderId) {
            this.unreadCounts[senderId] = (this.unreadCounts[senderId] || 0) + 1
            this.showNotification(`收到来自 "${senderName}" 的新消息`)
          } else {
            this.messages.push(message)
            if (this.autoTranslate) {
              this.translateSingleMessage(message)
            }
            this.scrollToBottom()
          }
        } catch (e) {
          console.warn('WS error', e)
        }
      }
      this.ws.onclose = (event) => {
        console.log('WebSocket 连接已关闭:', event)
        // this.showNotification('与服务器连接已断开。', 'error')
      }
      this.ws.onerror = (error) => {
        console.error('WebSocket 发生错误:', error)
      }
    }
  },
  beforeUnmount() {
    window.removeEventListener('resize', this.checkScreenSize)
    if (this.ws) this.ws.close()
    document.removeEventListener('keyup', this.handleGlobalKeyup)
  },
}
</script>

<style scoped>
.app-container {
  display: flex;
  height: 97vh;
  background-color: #f0f2f5;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  overflow: hidden; /* 防止整体滚动 */
}

/* 侧边栏过渡动画 */
.slide-side-enter-active,
.slide-side-leave-active {
  transition: all 0.3s ease;
}
.slide-side-enter-from,
.slide-side-leave-to {
  transform: translateX(-100%);
  opacity: 0;
  width: 0 !important;
  min-width: 0 !important;
}

.contacts-sidebar {
  width: 260px;
  background-color: #ffffff;
  border-right: 1px solid #e9e9eb;
  display: flex;
  flex-direction: column;
  flex-shrink: 0; /* 防止压缩 */
  z-index: 100;
}

.chat-container {
  position: relative;
  flex: 1;
  display: flex;
  flex-direction: column;
  height: 100%;
  background-color: #f7f8fa;
  min-width: 0; /* 防止子元素撑开flex */
}

/* --- 头部样式优化 --- */
.chat-header {
  padding: 10px 16px;
  border-bottom: 1px solid #e9e9eb;
  background-color: #ffffff;
  display: flex;
  align-items: center;
  justify-content: space-between;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.05);
  z-index: 10;
  height: 60px; /* 固定高度 */
  box-sizing: border-box;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 10px;
  flex: 1;
  min-width: 0; /* 允许文本截断 */
}

.header-right {
  display: flex;
  align-items: center;
}

.toggle-sidebar-btn {
  background: none;
  border: 1px solid #eee;
  border-radius: 4px;
  cursor: pointer;
  padding: 4px 8px;
  font-size: 14px;
  margin-right: 4px;
  color: #666;
}
.toggle-sidebar-btn:hover {
  background-color: #f5f5f5;
}

.chat-title {
  font-size: 18px;
  margin: 0;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.header-actions {
  display: flex;
  gap: 8px;
  margin-left: 8px;
}

.icon-btn {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 16px;
  padding: 4px;
}
.icon-btn:hover {
  background-color: #f0f0f0;
  border-radius: 4px;
}

.translation-controls {
  display: flex;
  align-items: center;
  gap: 8px;
  background-color: #f0f2f5;
  padding: 4px 10px;
  border-radius: 20px;
  margin-right: 10px;
}

/* --- 消息区域 --- */
.chat-messages {
  flex: 1;
  padding: 20px;
  overflow-y: auto;
  background-color: #f7f8fa;
  -webkit-overflow-scrolling: touch; /* iOS顺滑滚动 */
}

/* --- 底部输入区适配 --- */
.chat-input-wrapper {
  background-color: #ffffff;
  border-top: 1px solid #e9e9eb;
  flex-shrink: 0;
}

.ai-toolbar {
  display: flex;
  gap: 8px;
  padding: 8px 16px 0 16px;
  align-items: center;
  overflow-x: auto; /* 允许小屏幕横向滚动 */
  white-space: nowrap;
}

.ai-tool-btn {
  background-color: #f0f9eb;
  color: #42b983;
  border: 1px solid #e1f3d8;
  border-radius: 12px;
  padding: 4px 10px;
  font-size: 12px;
  cursor: pointer;
  transition: all 0.2s;
  flex-shrink: 0;
}

.chat-input-area {
  display: flex;
  padding: 12px 16px;
  gap: 10px;
  align-items: center;
}

.message-input {
  flex: 1;
  padding: 10px 14px;
  border: 1px solid #dcdfe6;
  border-radius: 20px;
  font-size: 14px;
  outline: none;
  min-width: 0;
}

.send-button {
  padding: 10px 20px;
  background-color: #42b983;
  color: white;
  border: none;
  border-radius: 20px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  white-space: nowrap;
}

/* --- 移动端适配 (Media Queries) --- */
@media (max-width: 768px) {
  .app-container {
    height: 100vh; /* 移动端占满全屏 */
    width: 100vw;
  }

  .contacts-sidebar {
    position: absolute; /* 绝对定位，覆盖在上方或独占 */
    width: 100%;
    height: 100%;
    z-index: 200;
    border-right: none;
  }

  /* 当在移动端模式且Sidebar隐藏时，让Chat Container占据全部 */
  .chat-container {
    width: 100%;
    height: 100%;
  }

  /* 调整Header */
  .chat-header {
    padding: 8px 10px;
  }

  .chat-title {
    font-size: 16px;
    max-width: 120px; /* 防止标题挤占按钮空间 */
  }

  /* 隐藏按钮文字，只留图标 */
  .mobile-hide-text .btn-text {
    display: none;
  }

  .summary-btn,
  .analysis-btn {
    padding: 6px; /* 减小padding */
    margin-left: 4px;
  }

  /* 调整输入区 */
  .chat-input-area {
    padding: 8px 10px;
  }

  .send-button {
    padding: 8px 16px;
  }

  .emoji-container {
    margin-right: 0;
  }

  /* 翻译控件在极小屏幕隐藏或简化 */
  .translation-controls {
    display: none; /* 移动端空间有限，暂时隐藏或移入设置菜单 */
  }

  .logout-btn {
    margin-left: 0;
    padding: 4px;
    font-size: 11px;
  }

  /* 消息气泡最大宽度调整 */
  .message-item {
    max-width: 88%;
  }
}

/* --- 原有样式保留 (以下为未修改的样式引用) --- */
/* (请保持原有的 .add-chat-item, .contact-avatar, .message-bubble 等样式，此处省略重复代码以节省空间，实际使用时请确保包含所有原样式) */
.add-chat-item {
  padding: 12px 16px;
  display: flex;
  align-items: center;
  cursor: pointer;
  border-bottom: 1px solid #f0f0f0;
  color: #42b983;
  font-weight: 500;
}
.add-chat-icon {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background-color: #ecfdf5;
  color: #42b983;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
  margin-right: 12px;
  flex-shrink: 0;
}
.add-chat-text {
  font-size: 15px;
}
.sidebar-header {
  padding: 16px;
  border-bottom: 1px solid #e9e9eb;
}
.sidebar-header h3 {
  margin: 0;
  font-size: 18px;
  color: #333;
}
.contacts-list {
  list-style: none;
  padding: 0;
  margin: 0;
  overflow-y: auto;
  flex: 1;
}
.contacts-list li {
  padding: 12px 16px;
  display: flex;
  align-items: center;
  cursor: pointer;
  border-bottom: 1px solid #f0f0f0;
  gap: 12px;
}
.contacts-list li:hover {
  background-color: #f5f5f5;
}
.contacts-list li.active {
  background-color: #e8f0fe;
  border-left: 3px solid #42b983;
}
.contact-avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background-color: #42b983;
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  flex-shrink: 0;
}
.contact-info {
  display: flex;
  flex-direction: column;
  justify-content: center;
  flex: 1;
  min-width: 0;
}
.unread-badge {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 18px;
  height: 18px;
  background-color: #ff4d4f;
  color: white;
  font-size: 12px;
  border-radius: 50%;
  margin-left: auto;
}
.contact-name {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 4px;
}
.nickname {
  font-size: 15px;
  font-weight: 600;
  color: #333;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.username {
  font-size: 12px;
  color: #999;
}
.last-message {
  font-size: 12px;
  color: #666;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.switch-label {
  display: flex;
  align-items: center;
  cursor: pointer;
  font-size: 13px;
  color: #555;
  gap: 6px;
}
.lang-select {
  border: 1px solid #ddd;
  border-radius: 4px;
  padding: 2px 6px;
  font-size: 12px;
  outline: none;
  background: white;
}
.empty-chat-hint {
  text-align: center;
  color: #999;
  padding-top: 30%;
  font-size: 14px;
}
.message-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}
.message-item {
  display: flex;
  max-width: 80%;
}
.self-message {
  align-self: flex-end;
}
.other-message {
  align-self: flex-start;
}
.message-bubble {
  padding: 10px 14px;
  border-radius: 18px;
  position: relative;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
  max-width: 100%;
  word-break: break-word;
}
.self-message .message-bubble {
  background-color: #42b983;
  color: white;
  border-top-right-radius: 4px;
}
.other-message .message-bubble {
  background-color: #ffffff;
  color: #333;
  border: 1px solid #e0e0e0;
  border-top-left-radius: 4px;
}
.message-sender-container {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 4px;
}
.message-sender {
  font-size: 12px;
  opacity: 0.8;
  font-weight: 500;
}
.message-time {
  font-size: 11px;
  margin-left: 8px;
  opacity: 0.7;
  color: #999;
}
.self-message .message-time {
  color: white;
}
.translation-content {
  margin-top: 8px;
  font-size: 14px;
}
.self-message .translation-content {
  color: #e6fffa;
}
.other-message .translation-content {
  color: #4a5568;
}
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
.divider {
  height: 1px;
  background-color: rgba(0, 0, 0, 0.1);
  margin: 6px 0;
}
.self-message .divider {
  background-color: rgba(255, 255, 255, 0.3);
}
.trans-icon {
  font-size: 12px;
  margin-right: 4px;
}
.translating-spinner {
  font-size: 12px;
  margin-top: 4px;
  opacity: 0.7;
  font-style: italic;
}
.manual-trans-btn {
  display: block;
  margin-top: 4px;
  font-size: 11px;
  color: #42b983;
  background: none;
  border: 1px solid #42b983;
  border-radius: 10px;
  padding: 1px 6px;
  cursor: pointer;
}
.ai-loading {
  font-size: 12px;
  color: #999;
  font-style: italic;
  margin-left: auto;
}
.current-user-item {
  display: flex;
  align-items: center;
  padding: 8px 12px;
  cursor: default;
}
.logout-btn {
  background: none;
  border: none;
  font-size: 12px;
  color: #666;
  cursor: pointer;
  padding: 4px 8px;
  border-radius: 4px;
  margin-left: 8px;
}
.ai-suggestion-box {
  padding: 10px 16px;
  background-color: #fffbe6;
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
.apply-btn,
.cancel-btn {
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
.cancel-btn {
  background-color: #ffffff;
  color: #666;
  border: 1px solid #ccc;
}
.summary-btn,
.analysis-btn {
  background-color: #f5f5f5;
  color: #606266;
  border: 1px solid #dcdfe6;
  border-radius: 4px;
  padding: 6px 12px;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.2s;
  margin-left: 10px;
}
.chat-summary-modal-overlay,
.analysis-modal-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}
.chat-summary-modal,
.analysis-modal {
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
  white-space: pre-wrap;
  font-size: 15px;
  line-height: 1.6;
  color: #303133;
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
}
.emoji-container {
  position: relative;
  align-self: center;
  flex-shrink: 0;
  margin-right: -5px;
}
.emoji-toggle-btn {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  padding: 0 10px;
  line-height: 1;
  color: #606266;
}
.emoji-picker-wrapper {
  position: absolute;
  bottom: 100%;
  left: -10px;
  margin-bottom: 10px;
  z-index: 20;
}
.chat-notification {
  position: absolute;
  top: 70px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 1000;
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
.slide-fade-enter-active,
.slide-fade-leave-active,
.slide-up-enter-active,
.slide-up-leave-active,
.modal-fade-enter-active,
.modal-fade-leave-active,
.slide-fade-fast-enter-active,
.slide-fade-fast-leave-active {
  transition: all 0.3s ease;
}
.slide-fade-enter,
.slide-fade-leave-to {
  transform: translate(-50%, -20px);
  opacity: 0;
}
.slide-up-enter,
.slide-up-leave-to {
  transform: translateY(100%);
  opacity: 0;
}
.modal-fade-enter,
.modal-fade-leave-to,
.slide-fade-fast-enter,
.slide-fade-fast-leave-to {
  opacity: 0;
}
.analysis-loading {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 100%;
  color: #666;
}
.loading-spinner {
  width: 40px;
  height: 40px;
  border: 4px solid #e1e1e1;
  border-top: 4px solid #1890ff;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin-bottom: 15px;
}
@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}
.dashboard-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  grid-template-rows: auto 1fr;
  gap: 20px;
}
.dashboard-card {
  background: white;
  border-radius: 10px;
  padding: 15px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
  height: 300px;
  display: flex;
  flex-direction: column;
}
.summary-card {
  grid-column: 1 / -1;
  height: auto;
  min-height: 100px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}
.card-title {
  font-size: 16px;
  font-weight: bold;
  margin-bottom: 10px;
  opacity: 0.9;
}
.ai-comment {
  font-size: 16px;
  line-height: 1.6;
  font-weight: 500;
}
.chart-container {
  width: 100%;
  height: 100%;
}
/* 语音气泡 */
.voice-content {
  display: flex;
  align-items: center;
  gap: 10px;
  cursor: pointer;
  min-width: 80px;
}
.voice-duration {
  font-size: 12px;
  opacity: 0.8;
}

/* 麦克风触发按钮 */
.mic-trigger-btn {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  padding: 0 5px;
  user-select: none;
}

/* 录音全屏遮罩 */
.recording-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.4);
  z-index: 3000;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  backdrop-filter: blur(2px);
}
.recording-card {
  background: white;
  padding: 30px;
  border-radius: 20px;
  text-align: center;
  width: 160px;
}
.status-icon {
  font-size: 50px;
  margin-bottom: 10px;
}
.status-hint {
  font-size: 14px;
  color: #666;
}
.status-hint.warning {
  color: #ff4d4f;
  font-weight: bold;
}

.record-zones {
  display: flex;
  gap: 40px;
  margin-top: 50px;
}
.zone {
  padding: 12px 20px;
  border-radius: 25px;
  background: rgba(255, 255, 255, 0.2);
  color: white;
  transition: 0.2s;
}
.zone.active {
  transform: scale(1.2);
  background: #ff4d4f;
}
.zone.right.active {
  background: #1890ff;
}

/* 呼吸灯效果 */
.pulse {
  animation: pulse-anim 1.5s infinite;
}
@keyframes pulse-anim {
  0% {
    transform: scale(1);
    opacity: 1;
  }
  50% {
    transform: scale(1.2);
    opacity: 0.7;
  }
  100% {
    transform: scale(1);
    opacity: 1;
  }
}
/* 语音消息气泡样式 */
.voice-player {
  display: flex;
  align-items: center;
  gap: 10px;
  cursor: pointer;
  padding: 5px 10px;
  background: #f0f2f5;
  border-radius: 8px;
  min-width: 60px;
  max-width: 200px;
  user-select: none;
}

.voice-player:active {
  background: #e1e4e8;
}

.voice-duration {
  font-size: 12px;
  color: #666;
}

.voice-icon {
  font-size: 16px;
}
/* 移动端看板适配 */
@media (max-width: 768px) {
  .dashboard-grid {
    grid-template-columns: 1fr; /* 变为单列 */
  }
}
</style>
