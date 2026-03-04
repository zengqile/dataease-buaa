<script lang="ts" setup>
import { ref, nextTick, onMounted } from 'vue'

// ── 消息类型 ────────────────────────────────────────────────────────────────
interface Message {
  id: number
  role: 'user' | 'bot'
  type: 'text' | 'voice'
  content: string
  time: string
  thinkingDone?: boolean
}

// ── 预设对话脚本（仅展示用） ─────────────────────────────────────────────────
const demoMessages: Message[] = [
  {
    id: 1,
    role: 'user',
    type: 'voice',
    content: '请查询北航2024年各专业录取分数线',
    time: '09:12'
  },
  {
    id: 2,
    role: 'bot',
    type: 'text',
    content: `好的，以下是**北京航空航天大学 2024 年各专业录取分数线（理科·普通批·北京市）**：

| 专业名称 | 最低分 | 平均分 | 最高分 | 录取人数 |
|:---|:---:|:---:|:---:|:---:|
| 飞行器设计与工程 | 682 | 691 | 703 | 45 |
| 航空航天工程 | 679 | 688 | 700 | 60 |
| 计算机科学与技术 | 691 | 698 | 712 | 80 |
| 软件工程 | 688 | 695 | 709 | 70 |
| 人工智能 | 693 | 701 | 715 | 50 |
| 电子信息工程 | 680 | 689 | 702 | 65 |
| 自动化 | 677 | 686 | 698 | 55 |
| 机械工程及自动化 | 672 | 681 | 693 | 50 |
| 材料科学与工程 | 668 | 677 | 690 | 40 |
| 工商管理 | 671 | 680 | 692 | 35 |

> 数据来源：北京航空航天大学 2024 年招生计划及录取结果公告`,
    time: '09:12',
    thinkingDone: true
  },
  {
    id: 3,
    role: 'user',
    type: 'text',
    content: '帮我分析一下各专业录取情况，哪些专业最热门？',
    time: '09:14'
  },
  {
    id: 4,
    role: 'bot',
    type: 'text',
    content: `根据 2024 年录取数据，以下是各专业热度分析：

**🔥 最热门专业 TOP 3**

1. **人工智能**（最低 693 分）— 分数线最高，竞争最激烈，反映出当前 AI 赛道的极高关注度，录取均分达 701 分。

2. **计算机科学与技术**（最低 691 分）— 传统强势专业，录取规模最大（80 人），持续供不应求。

3. **软件工程**（最低 688 分）— 与计算机协同发展，就业前景广阔，近年来分数线逐步攀升。

**✈️ 航空航天类专业**

- 飞行器设计与工程和航空航天工程分数线分别为 682 / 679 分，仍属超高竞争赛道，但相较计算机类略低，体现出专业特色吸引力。

**📊 整体趋势**

- 全校录取分数线普遍集中在 **668–715 分**区间，分数跨度约 47 分。
- 信息类专业（AI、CS、SE）形成高分聚集，航空航天工科类紧随其后。
- 材料科学与工程、工商管理录取门槛相对较低，入学机会相对更大。

**建议**：如果分数处于 680–690 区间，可重点考虑航空航天工程或电子信息工程，兼顾专业特色与录取可能性。`,
    time: '09:15',
    thinkingDone: true
  }
]

// ── 状态 ─────────────────────────────────────────────────────────────────────
const messages = ref<Message[]>([])
const inputText = ref('')
const inputMode = ref<'text' | 'voice'>('text')
const isRecording = ref(false)
const chatBody = ref<HTMLElement | null>(null)
const msgIdCounter = ref(100)

// 历史会话列表（仅展示）
const historyList = ref([
  { id: 1, title: '北航分数线查询', time: '今天', active: true },
  { id: 2, title: '北航各学院师资分析', time: '今天', active: false },
  { id: 3, title: '计算机专业就业前景', time: '昨天', active: false },
  { id: 4, title: '人工智能学科对比', time: '昨天', active: false },
  { id: 5, title: '2024年各校录取汇总', time: '本周', active: false }
])

// ── 方法 ─────────────────────────────────────────────────────────────────────
const scrollToBottom = async () => {
  await nextTick()
  if (chatBody.value) {
    chatBody.value.scrollTop = chatBody.value.scrollHeight
  }
}

const formatMarkdown = (text: string): string => {
  // 简单的 Markdown 渲染（表格、粗体、列表、引用）
  return text
    .replace(/\*\*(.+?)\*\*/g, '<strong>$1</strong>')
    .replace(/^#{1,3}\s+(.+)$/gm, '<h4>$1</h4>')
    .replace(/^>\s+(.+)$/gm, '<blockquote>$1</blockquote>')
    .replace(/\n\n/g, '</p><p>')
    .replace(/\n/g, '<br/>')
    .replace(/^(.*)$/, '<p>$1</p>')
    .replace(/<p><\/p>/g, '')
    .replace(
      /\|(.+)\|\n\|[-:| ]+\|\n((?:\|.+\|\n?)+)/g,
      (_m: string, header: string, rows: string) => {
        const ths = header
          .split('|')
          .filter(Boolean)
          .map((h: string) => `<th>${h.trim()}</th>`)
          .join('')
        const trs = rows
          .trim()
          .split('\n')
          .map((row: string) => {
            const tds = row
              .split('|')
              .filter(Boolean)
              .map((d: string) => `<td>${d.trim()}</td>`)
              .join('')
            return `<tr>${tds}</tr>`
          })
          .join('')
        return `<div class="md-table-wrap"><table><thead><tr>${ths}</tr></thead><tbody>${trs}</tbody></table></div>`
      }
    )
}

const toggleInputMode = () => {
  inputMode.value = inputMode.value === 'text' ? 'voice' : 'text'
  isRecording.value = false
}

const toggleRecording = () => {
  isRecording.value = !isRecording.value
}

const sendMessage = () => {
  if (!inputText.value.trim()) return
  const newMsg: Message = {
    id: ++msgIdCounter.value,
    role: 'user',
    type: 'text',
    content: inputText.value.trim(),
    time: new Date().toLocaleTimeString('zh-CN', { hour: '2-digit', minute: '2-digit' })
  }
  messages.value.push(newMsg)
  inputText.value = ''
  scrollToBottom()
}

const handleEnter = (e: KeyboardEvent) => {
  if (!e.shiftKey) {
    e.preventDefault()
    sendMessage()
  }
}

const selectHistory = (id: number) => {
  historyList.value.forEach(h => (h.active = h.id === id))
  if (id === 1) {
    messages.value = [...demoMessages]
  } else {
    messages.value = []
  }
  scrollToBottom()
}

// ── 初始化加载演示消息 ────────────────────────────────────────────────────────
onMounted(() => {
  messages.value = [...demoMessages]
  scrollToBottom()
})
</script>

<template>
  <div class="ai-chat-layout">
    <!-- ── 左侧历史会话 ──────────────────────────────────────────────── -->
    <aside class="chat-sidebar">
      <div class="sidebar-header">
        <span class="sidebar-title">
          <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor">
            <path
              d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm-1 14H9V8h2v8zm4 0h-2V8h2v8z"
            />
          </svg>
          AI 智能问答
        </span>
        <button class="new-chat-btn">
          <svg width="14" height="14" viewBox="0 0 24 24" fill="currentColor">
            <path d="M19 13H13v6h-2v-6H5v-2h6V5h2v6h6v2z" />
          </svg>
          新建对话
        </button>
      </div>

      <div class="history-list">
        <div class="history-group-label">今天</div>
        <div
          v-for="item in historyList.filter(h => h.time === '今天')"
          :key="item.id"
          class="history-item"
          :class="{ active: item.active }"
          @click="selectHistory(item.id)"
        >
          <svg class="history-icon" width="14" height="14" viewBox="0 0 24 24" fill="currentColor">
            <path d="M20 2H4c-1.1 0-2 .9-2 2v18l4-4h14c1.1 0 2-.9 2-2V4c0-1.1-.9-2-2-2z" />
          </svg>
          <span class="history-title">{{ item.title }}</span>
        </div>

        <div class="history-group-label">昨天</div>
        <div
          v-for="item in historyList.filter(h => h.time === '昨天')"
          :key="item.id"
          class="history-item"
          :class="{ active: item.active }"
          @click="selectHistory(item.id)"
        >
          <svg class="history-icon" width="14" height="14" viewBox="0 0 24 24" fill="currentColor">
            <path d="M20 2H4c-1.1 0-2 .9-2 2v18l4-4h14c1.1 0 2-.9 2-2V4c0-1.1-.9-2-2-2z" />
          </svg>
          <span class="history-title">{{ item.title }}</span>
        </div>

        <div class="history-group-label">本周</div>
        <div
          v-for="item in historyList.filter(h => h.time === '本周')"
          :key="item.id"
          class="history-item"
          :class="{ active: item.active }"
          @click="selectHistory(item.id)"
        >
          <svg class="history-icon" width="14" height="14" viewBox="0 0 24 24" fill="currentColor">
            <path d="M20 2H4c-1.1 0-2 .9-2 2v18l4-4h14c1.1 0 2-.9 2-2V4c0-1.1-.9-2-2-2z" />
          </svg>
          <span class="history-title">{{ item.title }}</span>
        </div>
      </div>

      <div class="sidebar-footer">
        <div class="model-tag">
          <svg width="14" height="14" viewBox="0 0 24 24" fill="currentColor">
            <path
              d="M13 2.05v2.02c3.95.49 7 3.85 7 7.93 0 3.21-1.81 6-4.5 7.54L13 17v5h5l-1.22-1.22C19.91 19.07 22 15.76 22 12c0-5.18-3.95-9.45-9-9.95zM11 2.05C5.95 2.55 2 6.82 2 12c0 3.76 2.09 7.07 5.22 8.78L6 22h5v-5l-2.5 2.47C6.81 18 5 15.21 5 12c0-4.08 3.05-7.44 7-7.93V2.05z"
            />
          </svg>
          DataEase AI · 数据分析版
        </div>
      </div>
    </aside>

    <!-- ── 右侧主聊天区 ───────────────────────────────────────────────── -->
    <main class="chat-main">
      <!-- 顶部标题栏 -->
      <div class="chat-topbar">
        <div class="chat-title">
          <span class="title-dot"></span>
          北航分数线查询
        </div>
        <div class="chat-mode-tabs">
          <span
            class="mode-tab"
            :class="{ active: inputMode === 'text' }"
            @click="inputMode = 'text'"
          >
            <svg width="14" height="14" viewBox="0 0 24 24" fill="currentColor">
              <path d="M2.5 4v3h5v12h3V7h5V4h-13zm19 5h-9v3h3v7h3v-7h3V9z" />
            </svg>
            文字问答
          </span>
          <span
            class="mode-tab"
            :class="{ active: inputMode === 'voice' }"
            @click="inputMode = 'voice'"
          >
            <svg width="14" height="14" viewBox="0 0 24 24" fill="currentColor">
              <path
                d="M12 1c-4.97 0-9 4.03-9 9v7c0 1.66 1.34 3 3 3h3v-8H5v-2c0-3.87 3.13-7 7-7s7 3.13 7 7v2h-4v8h3c1.66 0 3-1.34 3-3v-7c0-4.97-4.03-9-9-9z"
              />
            </svg>
            语音问答
          </span>
        </div>
      </div>

      <!-- 消息列表 -->
      <div class="chat-body" ref="chatBody">
        <!-- 空状态提示 -->
        <div v-if="messages.length === 0" class="empty-state">
          <div class="empty-icon">
            <svg width="64" height="64" viewBox="0 0 24 24" fill="currentColor">
              <path
                d="M20 2H4c-1.1 0-2 .9-2 2v18l4-4h14c1.1 0 2-.9 2-2V4c0-1.1-.9-2-2-2zm-2 12H6v-2h12v2zm0-3H6V9h12v2zm0-3H6V6h12v2z"
              />
            </svg>
          </div>
          <p class="empty-title">你好，我是 DataEase AI 助手</p>
          <p class="empty-desc">
            你可以向我提问关于数据集中的任何问题，我将基于数据为你提供精准分析。
          </p>
          <div class="quick-questions">
            <div class="quick-q" @click="inputText = '北航2024年录取分数线是多少？'">
              北航2024年录取分数线是多少？
            </div>
            <div class="quick-q" @click="inputText = '哪些专业竞争最激烈？'">
              哪些专业竞争最激烈？
            </div>
            <div class="quick-q" @click="inputText = '分析各专业录取趋势'">分析各专业录取趋势</div>
          </div>
        </div>

        <!-- 消息气泡 -->
        <template v-for="msg in messages" :key="msg.id">
          <!-- 用户消息 -->
          <div v-if="msg.role === 'user'" class="msg-row user-row">
            <div class="msg-content-wrap user-wrap">
              <!-- 语音标识 -->
              <div v-if="msg.type === 'voice'" class="voice-badge">
                <svg width="12" height="12" viewBox="0 0 24 24" fill="currentColor">
                  <path
                    d="M12 14c1.66 0 3-1.34 3-3V5c0-1.66-1.34-3-3-3S9 3.34 9 5v6c0 1.66 1.34 3 3 3zm-1-9c0-.55.45-1 1-1s1 .45 1 1v6c0 .55-.45 1-1 1s-1-.45-1-1V5zm6 6c0 2.76-2.24 5-5 5s-5-2.24-5-5H5c0 3.53 2.61 6.43 6 6.92V21h2v-3.08c3.39-.49 6-3.39 6-6.92h-2z"
                  />
                </svg>
                语音输入
              </div>
              <div class="msg-bubble user-bubble">{{ msg.content }}</div>
              <div class="msg-time">{{ msg.time }}</div>
            </div>
            <div class="avatar user-avatar">
              <svg width="20" height="20" viewBox="0 0 24 24" fill="currentColor">
                <path
                  d="M12 12c2.21 0 4-1.79 4-4s-1.79-4-4-4-4 1.79-4 4 1.79 4 4 4zm0 2c-2.67 0-8 1.34-8 4v2h16v-2c0-2.66-5.33-4-8-4z"
                />
              </svg>
            </div>
          </div>

          <!-- 机器人消息 -->
          <div v-else class="msg-row bot-row">
            <div class="avatar bot-avatar">
              <svg width="20" height="20" viewBox="0 0 24 24" fill="currentColor">
                <path
                  d="M20 9V7c0-1.1-.9-2-2-2h-3c0-1.66-1.34-3-3-3S9 3.34 9 5H6c-1.1 0-2 .9-2 2v2c-1.66 0-3 1.34-3 3s1.34 3 3 3v4c0 1.1.9 2 2 2h12c1.1 0 2-.9 2-2v-4c1.66 0 3-1.34 3-3s-1.34-3-3-3zm-2 11H6V7h12v13zm-9-6c-.83 0-1.5-.67-1.5-1.5S8.17 11 9 11s1.5.67 1.5 1.5S9.83 14 9 14zm6 0c-.83 0-1.5-.67-1.5-1.5S14.17 11 15 11s1.5.67 1.5 1.5S15.83 14 15 14z"
                />
              </svg>
            </div>
            <div class="msg-content-wrap bot-wrap">
              <div class="bot-name">DataEase AI</div>
              <div class="msg-bubble bot-bubble" v-html="formatMarkdown(msg.content)"></div>
              <div class="msg-time">{{ msg.time }}</div>
              <!-- 操作按钮 -->
              <div class="msg-actions">
                <button class="action-btn" title="复制">
                  <svg width="12" height="12" viewBox="0 0 24 24" fill="currentColor">
                    <path
                      d="M16 1H4c-1.1 0-2 .9-2 2v14h2V3h12V1zm3 4H8c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h11c1.1 0 2-.9 2-2V7c0-1.1-.9-2-2-2zm0 16H8V7h11v14z"
                    />
                  </svg>
                  复制
                </button>
                <button class="action-btn" title="朗读">
                  <svg width="12" height="12" viewBox="0 0 24 24" fill="currentColor">
                    <path
                      d="M3 9v6h4l5 5V4L7 9H3zm13.5 3c0-1.77-1.02-3.29-2.5-4.03v8.05c1.48-.73 2.5-2.25 2.5-4.02zM14 3.23v2.06c2.89.86 5 3.54 5 6.71s-2.11 5.85-5 6.71v2.06c4.01-.91 7-4.49 7-8.77s-2.99-7.86-7-8.77z"
                    />
                  </svg>
                  朗读
                </button>
                <button class="action-btn like-btn" title="有帮助">
                  <svg width="12" height="12" viewBox="0 0 24 24" fill="currentColor">
                    <path
                      d="M1 21h4V9H1v12zm22-11c0-1.1-.9-2-2-2h-6.31l.95-4.57.03-.32c0-.41-.17-.79-.44-1.06L14.17 1 7.59 7.59C7.22 7.95 7 8.45 7 9v10c0 1.1.9 2 2 2h9c.83 0 1.54-.5 1.84-1.22l3.02-7.05c.09-.23.14-.47.14-.73v-2z"
                    />
                  </svg>
                </button>
              </div>
            </div>
          </div>
        </template>
      </div>

      <!-- ── 输入区 ─────────────────────────────────────────────────── -->
      <div class="chat-input-area">
        <!-- 文字输入模式 -->
        <div v-if="inputMode === 'text'" class="text-input-wrap">
          <div class="input-toolbar">
            <button class="toolbar-btn" title="切换语音输入" @click="toggleInputMode">
              <svg width="16" height="16" viewBox="0 0 24 24" fill="currentColor">
                <path
                  d="M12 14c1.66 0 3-1.34 3-3V5c0-1.66-1.34-3-3-3S9 3.34 9 5v6c0 1.66 1.34 3 3 3zm-1-9c0-.55.45-1 1-1s1 .45 1 1v6c0 .55-.45 1-1 1s-1-.45-1-1V5zm6 6c0 2.76-2.24 5-5 5s-5-2.24-5-5H5c0 3.53 2.61 6.43 6 6.92V21h2v-3.08c3.39-.49 6-3.39 6-6.92h-2z"
                />
              </svg>
              语音
            </button>
            <span class="toolbar-hint">Shift+Enter 换行 · Enter 发送</span>
          </div>
          <div class="input-row">
            <textarea
              v-model="inputText"
              class="chat-textarea"
              placeholder="基于数据内容提问，例如：北航哪个专业录取人数最多？"
              rows="3"
              @keydown.enter="handleEnter"
            ></textarea>
            <button class="send-btn" :class="{ active: inputText.trim() }" @click="sendMessage">
              <svg width="20" height="20" viewBox="0 0 24 24" fill="currentColor">
                <path d="M2.01 21L23 12 2.01 3 2 10l15 2-15 2z" />
              </svg>
            </button>
          </div>
        </div>

        <!-- 语音输入模式 -->
        <div v-else class="voice-input-wrap">
          <button class="toolbar-btn text-mode-btn" @click="toggleInputMode">
            <svg width="16" height="16" viewBox="0 0 24 24" fill="currentColor">
              <path d="M2.5 4v3h5v12h3V7h5V4h-13zm19 5h-9v3h3v7h3v-7h3V9z" />
            </svg>
            文字
          </button>
          <div class="voice-center">
            <div class="voice-btn" :class="{ recording: isRecording }" @click="toggleRecording">
              <div class="voice-ripple" v-if="isRecording"></div>
              <div class="voice-ripple-2" v-if="isRecording"></div>
              <svg width="32" height="32" viewBox="0 0 24 24" fill="currentColor">
                <path
                  d="M12 14c1.66 0 3-1.34 3-3V5c0-1.66-1.34-3-3-3S9 3.34 9 5v6c0 1.66 1.34 3 3 3zm-1-9c0-.55.45-1 1-1s1 .45 1 1v6c0 .55-.45 1-1 1s-1-.45-1-1V5zm6 6c0 2.76-2.24 5-5 5s-5-2.24-5-5H5c0 3.53 2.61 6.43 6 6.92V21h2v-3.08c3.39-.49 6-3.39 6-6.92h-2z"
                />
              </svg>
            </div>
            <p class="voice-hint" v-if="!isRecording">点击麦克风开始语音输入</p>
            <p class="voice-hint recording-hint" v-else>
              <span class="dot-flash">●</span> 正在聆听，请说话...
            </p>
            <div v-if="isRecording" class="voice-wave">
              <span
                v-for="i in 12"
                :key="i"
                class="wave-bar"
                :style="{ animationDelay: `${i * 0.08}s` }"
              ></span>
            </div>
          </div>
        </div>

        <div class="input-footer-tip">DataEase AI 基于当前数据集内容作答，结果仅供参考</div>
      </div>
    </main>
  </div>
</template>

<style lang="less" scoped>
// ── 变量 ──────────────────────────────────────────────────────────────────────
@bg-dark: #0d1117;
@bg-sidebar: #161b22;
@bg-main: #f0f2f5;
@bg-card: #ffffff;
@primary: #3371ff;
@primary-light: rgba(51, 113, 255, 0.12);
@text-primary: #1f2329;
@text-secondary: #646a73;
@text-muted: #8f959e;
@border: #e4e7ed;
@bot-bubble-bg: #ffffff;
@user-bubble-bg: #3371ff;
@radius: 12px;

// ── 布局 ──────────────────────────────────────────────────────────────────────
.ai-chat-layout {
  display: flex;
  height: calc(100vh - 56px);
  background: @bg-main;
  overflow: hidden;
}

// ── 左侧侧边栏 ───────────────────────────────────────────────────────────────
.chat-sidebar {
  width: 260px;
  min-width: 260px;
  background: @bg-sidebar;
  display: flex;
  flex-direction: column;
  border-right: 1px solid rgba(255, 255, 255, 0.06);

  .sidebar-header {
    padding: 20px 16px 12px;
    display: flex;
    align-items: center;
    justify-content: space-between;
    border-bottom: 1px solid rgba(255, 255, 255, 0.06);

    .sidebar-title {
      display: flex;
      align-items: center;
      gap: 8px;
      color: #e6edf3;
      font-size: 15px;
      font-weight: 600;
      svg {
        color: @primary;
      }
    }

    .new-chat-btn {
      display: flex;
      align-items: center;
      gap: 4px;
      padding: 5px 10px;
      background: @primary;
      border: none;
      border-radius: 6px;
      color: #fff;
      font-size: 12px;
      cursor: pointer;
      transition: background 0.2s;
      &:hover {
        background: #2b5fd9;
      }
    }
  }

  .history-list {
    flex: 1;
    overflow-y: auto;
    padding: 8px 0;

    &::-webkit-scrollbar {
      width: 4px;
    }
    &::-webkit-scrollbar-track {
      background: transparent;
    }
    &::-webkit-scrollbar-thumb {
      background: rgba(255, 255, 255, 0.1);
      border-radius: 2px;
    }

    .history-group-label {
      padding: 8px 16px 4px;
      color: rgba(255, 255, 255, 0.35);
      font-size: 11px;
      font-weight: 500;
      letter-spacing: 0.5px;
      text-transform: uppercase;
    }

    .history-item {
      display: flex;
      align-items: center;
      gap: 8px;
      padding: 9px 16px;
      cursor: pointer;
      border-radius: 6px;
      margin: 0 8px 2px;
      transition: background 0.15s;

      .history-icon {
        color: rgba(255, 255, 255, 0.35);
        flex-shrink: 0;
      }
      .history-title {
        color: rgba(255, 255, 255, 0.6);
        font-size: 13px;
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
      }

      &:hover {
        background: rgba(255, 255, 255, 0.06);
        .history-title {
          color: rgba(255, 255, 255, 0.85);
        }
      }

      &.active {
        background: rgba(51, 113, 255, 0.18);
        .history-icon {
          color: @primary;
        }
        .history-title {
          color: #fff;
          font-weight: 500;
        }
      }
    }
  }

  .sidebar-footer {
    padding: 14px 16px;
    border-top: 1px solid rgba(255, 255, 255, 0.06);

    .model-tag {
      display: flex;
      align-items: center;
      gap: 6px;
      color: rgba(255, 255, 255, 0.4);
      font-size: 11px;
      svg {
        color: #00d6b9;
      }
    }
  }
}

// ── 右侧主区域 ───────────────────────────────────────────────────────────────
.chat-main {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background: @bg-main;
}

// 顶部栏
.chat-topbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 14px 28px;
  background: @bg-card;
  border-bottom: 1px solid @border;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.04);

  .chat-title {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 15px;
    font-weight: 600;
    color: @text-primary;

    .title-dot {
      width: 8px;
      height: 8px;
      border-radius: 50%;
      background: #00d6b9;
      box-shadow: 0 0 6px rgba(0, 214, 185, 0.6);
    }
  }

  .chat-mode-tabs {
    display: flex;
    gap: 4px;
    background: #f0f2f5;
    border-radius: 8px;
    padding: 3px;

    .mode-tab {
      display: flex;
      align-items: center;
      gap: 5px;
      padding: 5px 12px;
      border-radius: 6px;
      font-size: 13px;
      color: @text-secondary;
      cursor: pointer;
      transition: all 0.2s;

      &.active {
        background: @bg-card;
        color: @primary;
        font-weight: 500;
        box-shadow: 0 1px 4px rgba(0, 0, 0, 0.08);
      }

      &:hover:not(.active) {
        color: @text-primary;
      }
    }
  }
}

// 消息区
.chat-body {
  flex: 1;
  overflow-y: auto;
  padding: 24px 28px;
  display: flex;
  flex-direction: column;
  gap: 20px;

  &::-webkit-scrollbar {
    width: 6px;
  }
  &::-webkit-scrollbar-track {
    background: transparent;
  }
  &::-webkit-scrollbar-thumb {
    background: rgba(0, 0, 0, 0.12);
    border-radius: 3px;
  }
}

// 空状态
.empty-state {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 60px 20px;
  text-align: center;

  .empty-icon {
    width: 80px;
    height: 80px;
    background: linear-gradient(135deg, @primary, #00d6b9);
    border-radius: 20px;
    display: flex;
    align-items: center;
    justify-content: center;
    margin-bottom: 20px;
    svg {
      color: white;
    }
    box-shadow: 0 8px 24px rgba(51, 113, 255, 0.3);
  }

  .empty-title {
    font-size: 20px;
    font-weight: 600;
    color: @text-primary;
    margin: 0 0 8px;
  }

  .empty-desc {
    font-size: 14px;
    color: @text-secondary;
    max-width: 400px;
    line-height: 1.6;
    margin: 0 0 28px;
  }

  .quick-questions {
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
    justify-content: center;

    .quick-q {
      padding: 8px 16px;
      background: @bg-card;
      border: 1px solid @border;
      border-radius: 20px;
      font-size: 13px;
      color: @text-secondary;
      cursor: pointer;
      transition: all 0.2s;
      &:hover {
        border-color: @primary;
        color: @primary;
        background: @primary-light;
      }
    }
  }
}

// 消息行
.msg-row {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  max-width: 900px;

  &.user-row {
    flex-direction: row-reverse;
    align-self: flex-end;
    max-width: 70%;
  }

  &.bot-row {
    align-self: flex-start;
    max-width: 80%;
  }
}

// 头像
.avatar {
  width: 36px;
  height: 36px;
  border-radius: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;

  &.user-avatar {
    background: linear-gradient(135deg, @primary, #1a56db);
    svg {
      color: white;
    }
  }

  &.bot-avatar {
    background: linear-gradient(135deg, #0d1117, #1e2738);
    border: 1px solid rgba(51, 113, 255, 0.3);
    svg {
      color: @primary;
    }
  }
}

// 消息内容容器
.msg-content-wrap {
  display: flex;
  flex-direction: column;
  gap: 4px;

  &.user-wrap {
    align-items: flex-end;
  }
  &.bot-wrap {
    align-items: flex-start;
  }

  .bot-name {
    font-size: 12px;
    font-weight: 500;
    color: @text-muted;
    margin-bottom: 2px;
  }

  .msg-time {
    font-size: 11px;
    color: @text-muted;
    margin-top: 2px;
  }
}

// 语音标识
.voice-badge {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 3px 8px;
  background: rgba(51, 113, 255, 0.12);
  border-radius: 12px;
  font-size: 11px;
  color: @primary;
  margin-bottom: 2px;
  svg {
    color: @primary;
  }
}

// 气泡
.msg-bubble {
  padding: 12px 16px;
  border-radius: @radius;
  font-size: 14px;
  line-height: 1.65;
  word-break: break-word;

  &.user-bubble {
    background: @user-bubble-bg;
    color: white;
    border-bottom-right-radius: 4px;
  }

  &.bot-bubble {
    background: @bot-bubble-bg;
    color: @text-primary;
    border-bottom-left-radius: 4px;
    box-shadow: 0 2px 12px rgba(0, 0, 0, 0.06);
    border: 1px solid @border;

    :deep(strong) {
      font-weight: 600;
      color: #1a1a2e;
    }
    :deep(h4) {
      font-size: 14px;
      font-weight: 600;
      color: @text-primary;
      margin: 12px 0 6px;
    }
    :deep(p) {
      margin: 6px 0;
    }
    :deep(br) {
      line-height: 1.8;
    }
    :deep(blockquote) {
      border-left: 3px solid @primary;
      padding: 6px 12px;
      background: @primary-light;
      border-radius: 0 6px 6px 0;
      color: @text-secondary;
      font-size: 13px;
      margin: 8px 0 0;
    }

    :deep(.md-table-wrap) {
      overflow-x: auto;
      margin: 10px 0;

      table {
        width: 100%;
        border-collapse: collapse;
        font-size: 13px;

        th {
          background: #f0f4ff;
          color: @primary;
          padding: 8px 12px;
          text-align: left;
          font-weight: 600;
          border: 1px solid #dce5ff;
        }

        td {
          padding: 8px 12px;
          border: 1px solid @border;
          color: @text-primary;
        }

        tr:nth-child(even) td {
          background: #fafbff;
        }
        tr:hover td {
          background: @primary-light;
        }
      }
    }
  }
}

// 消息操作按钮
.msg-actions {
  display: flex;
  gap: 4px;
  margin-top: 4px;
  opacity: 0;
  transition: opacity 0.2s;

  .action-btn {
    display: flex;
    align-items: center;
    gap: 3px;
    padding: 3px 8px;
    background: transparent;
    border: 1px solid @border;
    border-radius: 6px;
    color: @text-muted;
    font-size: 11px;
    cursor: pointer;
    transition: all 0.15s;

    &:hover {
      background: @primary-light;
      border-color: @primary;
      color: @primary;
    }
  }

  .like-btn {
    padding: 3px 6px;
  }
}

.bot-wrap:hover .msg-actions {
  opacity: 1;
}

// ── 输入区 ────────────────────────────────────────────────────────────────────
.chat-input-area {
  padding: 16px 28px 20px;
  background: @bg-card;
  border-top: 1px solid @border;

  .input-footer-tip {
    text-align: center;
    font-size: 11px;
    color: @text-muted;
    margin-top: 8px;
  }
}

// 文字输入
.text-input-wrap {
  .input-toolbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 8px;

    .toolbar-btn {
      display: flex;
      align-items: center;
      gap: 5px;
      padding: 5px 12px;
      background: #f0f2f5;
      border: 1px solid @border;
      border-radius: 6px;
      color: @text-secondary;
      font-size: 12px;
      cursor: pointer;
      transition: all 0.15s;
      &:hover {
        background: @primary-light;
        border-color: @primary;
        color: @primary;
      }
    }

    .toolbar-hint {
      font-size: 11px;
      color: @text-muted;
    }
  }

  .input-row {
    display: flex;
    gap: 10px;
    align-items: flex-end;

    .chat-textarea {
      flex: 1;
      padding: 12px 14px;
      background: #f8f9fc;
      border: 1px solid @border;
      border-radius: 10px;
      font-size: 14px;
      color: @text-primary;
      resize: none;
      outline: none;
      font-family: inherit;
      line-height: 1.6;
      transition: border-color 0.2s;

      &:focus {
        border-color: @primary;
        background: white;
      }
      &::placeholder {
        color: @text-muted;
      }
    }

    .send-btn {
      width: 44px;
      height: 44px;
      background: #e4e7ed;
      border: none;
      border-radius: 10px;
      display: flex;
      align-items: center;
      justify-content: center;
      cursor: pointer;
      transition: all 0.2s;
      flex-shrink: 0;
      svg {
        color: #b0b7c3;
      }

      &.active {
        background: @primary;
        box-shadow: 0 4px 12px rgba(51, 113, 255, 0.35);
        svg {
          color: white;
        }
      }

      &.active:hover {
        background: #2b5fd9;
      }
    }
  }
}

// 语音输入
.voice-input-wrap {
  display: flex;
  align-items: center;
  gap: 20px;
  padding: 10px 0;

  .text-mode-btn {
    display: flex;
    align-items: center;
    gap: 5px;
    padding: 6px 14px;
    background: #f0f2f5;
    border: 1px solid @border;
    border-radius: 6px;
    color: @text-secondary;
    font-size: 12px;
    cursor: pointer;
    white-space: nowrap;
    transition: all 0.15s;
    &:hover {
      background: @primary-light;
      border-color: @primary;
      color: @primary;
    }
  }

  .voice-center {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 12px;
  }

  .voice-btn {
    position: relative;
    width: 64px;
    height: 64px;
    border-radius: 50%;
    background: linear-gradient(135deg, @primary, #1a56db);
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    box-shadow: 0 4px 16px rgba(51, 113, 255, 0.35);
    transition: all 0.2s;
    svg {
      color: white;
    }

    &:hover {
      transform: scale(1.05);
    }

    &.recording {
      background: linear-gradient(135deg, #ff4757, #c0392b);
      box-shadow: 0 4px 16px rgba(255, 71, 87, 0.4);
    }

    .voice-ripple,
    .voice-ripple-2 {
      position: absolute;
      width: 100%;
      height: 100%;
      border-radius: 50%;
      border: 2px solid rgba(255, 71, 87, 0.5);
      animation: ripple 1.4s infinite;
    }

    .voice-ripple-2 {
      animation-delay: 0.7s;
    }
  }

  .voice-hint {
    font-size: 13px;
    color: @text-secondary;
    margin: 0;
  }

  .recording-hint {
    color: #ff4757;
    display: flex;
    align-items: center;
    gap: 6px;

    .dot-flash {
      animation: flash 0.8s infinite;
      font-size: 10px;
    }
  }

  .voice-wave {
    display: flex;
    align-items: center;
    gap: 3px;
    height: 32px;

    .wave-bar {
      display: inline-block;
      width: 3px;
      height: 8px;
      background: @primary;
      border-radius: 2px;
      animation: wave 0.8s ease-in-out infinite alternate;
    }
  }
}

// ── 动画 ──────────────────────────────────────────────────────────────────────
@keyframes ripple {
  0% {
    transform: scale(1);
    opacity: 0.8;
  }
  100% {
    transform: scale(1.8);
    opacity: 0;
  }
}

@keyframes flash {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0.2;
  }
}

@keyframes wave {
  0% {
    height: 4px;
  }
  100% {
    height: 28px;
  }
}
</style>
