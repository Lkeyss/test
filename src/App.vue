<template>
  <div class="app-layout">
    <!-- 左侧会话列表 -->
    <div class="sidebar">
      <div class="sidebar-header">
        <span class="sidebar-title">会话列表</span>
        <button class="new-chat-btn" @click="createNewSession" title="新建会话">＋ 新会话</button>
      </div>
      <div class="session-list">
        <div
          v-for="s in sessionList"
          :key="s.sessionId"
          class="session-item"
          :class="{ active: s.sessionId === sessionId }"
          @click="switchSession(s)"
        >
          <div class="session-title">{{ s.title || '新会话' }}</div>
          <div class="session-time">{{ formatTime(s.updateTime) }}</div>
          <button class="del-btn" @click.stop="deleteSession(s.sessionId)" title="删除">✕</button>
        </div>
        <div v-if="sessionList.length === 0" class="no-session">暂无会话</div>
      </div>
    </div>

    <!-- 右侧聊天区域 -->
    <div class="chat-app">
      <!-- 头部 -->
      <div class="chat-header">
        <div class="avatar">🤖</div>
        <div class="header-info">
          <div class="name">无敌小P</div>
          <div class="status online">在线</div>
        </div>
      </div>

      <!-- 消息列表 -->
      <div class="chat-messages" ref="messagesContainer">
        <div v-if="messages.length === 0" class="welcome">
          <div class="welcome-icon">👋</div>
          <div class="welcome-text">您好！我是小智，您的专属购物助手</div>
          <div class="welcome-sub">可以帮您查询订单、处理售后、解答疑问</div>
          <div class="quick-actions">
            <button v-for="q in quickQuestions" :key="q" @click="sendQuick(q)">{{ q }}</button>
          </div>
        </div>

        <div v-for="(msg, index) in messages" :key="index" class="message" :class="msg.role">
          <div v-if="msg.role === 'assistant'" class="msg-avatar">🤖</div>
          <div class="msg-content">
            <div v-if="msg.toolStatus" class="tool-status">
              <span class="tool-spinner">⟳</span> {{ msg.toolStatus }}
            </div>
            <!-- 卡片消息 -->
            <CardRenderer
              v-if="msg.type === 'card'"
              :card-type="msg.cardType"
              :payload="msg.payload"
              :session-id="sessionId"
              :user-id="USER_ID"
              :msg-id="msg.msgId || ''"
              :init-submitted="msg.submitted"
              @submitted="onCardSubmitted(msg, $event)"
            />
            <!-- 普通文本消息 -->
            <div v-else class="msg-text" v-html="formatMessage(msg.content)"></div>
            <div class="msg-time">{{ msg.time }}</div>
          </div>
          <div v-if="msg.role === 'user'" class="msg-avatar user-avatar">👤</div>
        </div>

        <div v-if="isStreaming" class="message assistant">
          <div class="msg-avatar">🤖</div>
          <div class="msg-content">
            <div v-if="currentToolStatus" class="tool-status">
              <span class="tool-spinner spinning">⟳</span> {{ currentToolStatus }}
            </div>
            <div class="msg-text streaming-text">
              {{ streamingBuffer }}<span class="cursor">▋</span>
            </div>
          </div>
        </div>
      </div>

      <!-- 输入框 -->
      <div class="chat-input-area">
        <textarea
          ref="inputRef"
          v-model="inputText"
          placeholder="请描述您的问题，例如：查询订单ORD001的物流"
          :disabled="isStreaming"
          @keydown.enter.prevent="handleEnter"
          rows="1"
          @input="autoResize"
        ></textarea>
        <button class="send-btn" :disabled="!inputText.trim() || isStreaming" @click="sendMessage">
          <span v-if="!isStreaming">发送</span>
          <span v-else class="stop-icon" @click.stop="stopStreaming">■ 停止</span>
        </button>
      </div>
      <div class="input-hint">
        Enter 发送 · Shift+Enter 换行
        <span v-if="locationStatus === 'locating'" class="loc-status locating">⟳ 定位中...</span>
        <span v-else-if="locationStatus === 'ok'" class="loc-status ok" :title="`GCJ-02: ${userLocation?.lat}, ${userLocation?.lng}`">
          📍 已定位 ({{ userLocation?.lat?.toFixed(3) }}, {{ userLocation?.lng?.toFixed(3) }})
        </span>
        <span v-else-if="locationStatus === 'fail'" class="loc-status fail" title="请手动告知城市">
          📍 定位失败，请告知所在城市
        </span>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick, onMounted, onUnmounted } from 'vue'
import CardRenderer from './components/CardRenderer.vue'

const BASE_URL = 'http://localhost:8080'
const USER_ID = 'demo_user'
const DEFAULT_ERROR_MSG = '抱歉，服务出现异常，请稍后重试或联系人工客服 🙏'

// ── 状态 ─────────────────────────────────────────────────────
const messages = ref([])
const inputText = ref('')
const isStreaming = ref(false)
const streamingBuffer = ref('')
const currentToolStatus = ref('')
const messagesContainer = ref(null)
const inputRef = ref(null)
const sessionId = ref('')
const sessionList = ref([])
let currentReader = null

// 用户位置（发送消息时携带，用于附近超充站查询）
const userLocation = ref(null)
const locationStatus = ref('') // 'locating' | 'ok' | 'fail' | ''

// ── WGS-84 → GCJ-02 坐标转换（高德/百度地图使用 GCJ-02）──────
// 浏览器 navigator.geolocation 返回 WGS-84，直接传给高德会有偏差
const PI = Math.PI
function _transformLat(x, y) {
  let ret = -100.0 + 2.0*x + 3.0*y + 0.2*y*y + 0.1*x*y + 0.2*Math.sqrt(Math.abs(x))
  ret += (20.0*Math.sin(6.0*x*PI) + 20.0*Math.sin(2.0*x*PI)) * 2.0/3.0
  ret += (20.0*Math.sin(y*PI) + 40.0*Math.sin(y/3.0*PI)) * 2.0/3.0
  ret += (160.0*Math.sin(y/12.0*PI) + 320*Math.sin(y*PI/30.0)) * 2.0/3.0
  return ret
}
function _transformLng(x, y) {
  let ret = 300.0 + x + 2.0*y + 0.1*x*x + 0.1*x*y + 0.1*Math.sqrt(Math.abs(x))
  ret += (20.0*Math.sin(6.0*x*PI) + 20.0*Math.sin(2.0*x*PI)) * 2.0/3.0
  ret += (20.0*Math.sin(x*PI) + 40.0*Math.sin(x/3.0*PI)) * 2.0/3.0
  ret += (150.0*Math.sin(x/12.0*PI) + 300.0*Math.sin(x/30.0*PI)) * 2.0/3.0
  return ret
}
function wgs84ToGcj02(lat, lng) {
  const a = 6378245.0, ee = 0.00669342162296594323
  let dLat = _transformLat(lng - 105.0, lat - 35.0)
  let dLng = _transformLng(lng - 105.0, lat - 35.0)
  const radLat = lat / 180.0 * PI
  let magic = Math.sin(radLat)
  magic = 1 - ee * magic * magic
  const sqrtMagic = Math.sqrt(magic)
  dLat = (dLat * 180.0) / ((a * (1 - ee)) / (magic * sqrtMagic) * PI)
  dLng = (dLng * 180.0) / (a / sqrtMagic * Math.cos(radLat) * PI)
  return { lat: +(lat + dLat).toFixed(6), lng: +(lng + dLng).toFixed(6) }
}

let _watchId = null

// 两点间距离（km），Haversine 公式
function haversineKm(lat1, lng1, lat2, lng2) {
  const R = 6371
  const dLat = (lat2 - lat1) * PI / 180
  const dLng = (lng2 - lng1) * PI / 180
  const a = Math.sin(dLat/2)**2 + Math.cos(lat1*PI/180) * Math.cos(lat2*PI/180) * Math.sin(dLng/2)**2
  return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a))
}

// IP 定位：依次尝试多个免费接口（首选支持 CORS 的服务），返回 GCJ-02 坐标
async function locateByIP() {
  // 1) ipwho.is - 支持 HTTPS + CORS，免费
  try {
    const res = await fetch('https://ipwho.is/', { signal: AbortSignal.timeout(5000) })
    const data = await res.json()
    if (data && data.success !== false && data.latitude && data.longitude) {
      const gcj = wgs84ToGcj02(Number(data.latitude), Number(data.longitude))
      console.log(`[IP定位] ipwho.is GCJ-02: (${gcj.lat}, ${gcj.lng})`)
      return gcj
    }
  } catch (e) {
    console.warn('[IP定位] ipwho.is 失败:', e.message)
  }

  // 2) ipapi.co - 支持 HTTPS，通常允许 CORS
  try {
    const res = await fetch('https://ipapi.co/json/', { signal: AbortSignal.timeout(5000) })
    const data = await res.json()
    if (data && data.latitude && data.longitude) {
      const gcj = wgs84ToGcj02(Number(data.latitude), Number(data.longitude))
      console.log(`[IP定位] ipapi.co GCJ-02: (${gcj.lat}, ${gcj.lng})`)
      return gcj
    }
  } catch (e) {
    console.warn('[IP定位] ipapi.co 失败:', e.message)
  }

  // 3) ipinfo.io - 作为最后回退（注意部分情况下需要 token 或受限）
  try {
    const res = await fetch('https://ipinfo.io/json', { signal: AbortSignal.timeout(5000) })
    const data = await res.json()
    if (data && data.loc) {
      const [lat, lng] = data.loc.split(',').map(Number)
      const gcj = wgs84ToGcj02(lat, lng)
      console.log(`[IP定位] ipinfo.io GCJ-02: (${gcj.lat}, ${gcj.lng})`)
      return gcj
    }
  } catch (e) {
    console.warn('[IP定位] ipinfo.io 失败:', e.message)
  }

  return null
}

// GPS 获取一次定位，返回原始 WGS-84 或 null
function getGPSOnce() {
  return new Promise(resolve => {
    if (!navigator.geolocation) return resolve(null)
    navigator.geolocation.getCurrentPosition(
      pos => resolve({ lat: pos.coords.latitude, lng: pos.coords.longitude, accuracy: pos.coords.accuracy }),
      () => resolve(null),
      { enableHighAccuracy: true, timeout: 8000, maximumAge: 0 }
    )
  })
}

async function requestUserLocation() {
  locationStatus.value = 'locating'

  // 并行获取 GPS 和 IP，以 IP 作为可信基准校验 GPS 是否合理
  const [gpsWgs, ipGcj] = await Promise.all([getGPSOnce(), locateByIP()])

  if (gpsWgs) {
    const gpsGcj = wgs84ToGcj02(gpsWgs.lat, gpsWgs.lng)
    if (ipGcj) {
      const dist = haversineKm(gpsGcj.lat, gpsGcj.lng, ipGcj.lat, ipGcj.lng)
      console.log(`[定位校验] GPS与IP距离: ${dist.toFixed(0)}km`)
      if (dist < 300) {
        // GPS 与 IP 位置吻合，GPS 更精确，采用 GPS
        userLocation.value = gpsGcj
        locationStatus.value = 'ok'
        console.log(`[定位] 采用GPS: (${gpsGcj.lat}, ${gpsGcj.lng})`)
        return
      } else {
        // GPS 与 IP 偏差超过 300km，Windows 定位数据库错误，采用 IP
        console.warn(`[定位] GPS偏差 ${dist.toFixed(0)}km，采用IP定位`)
        userLocation.value = ipGcj
        locationStatus.value = 'ok'
        return
      }
    } else {
      // 无 IP 基准，直接用 GPS
      userLocation.value = gpsGcj
      locationStatus.value = 'ok'
      console.log(`[定位] 采用GPS(无IP基准): (${gpsGcj.lat}, ${gpsGcj.lng})`)
      return
    }
  }

  // GPS 不可用，直接用 IP
  if (ipGcj) {
    userLocation.value = ipGcj
    locationStatus.value = 'ok'
    console.log(`[定位] 采用IP定位: (${ipGcj.lat}, ${ipGcj.lng})`)
  } else {
    locationStatus.value = 'fail'
  }
}

function stopWatchLocation() {
  // getCurrentPosition 不需要清理，_watchId 保留供将来使用
  _watchId = null
}

const quickQuestions = ['查询订单ORD001', '我的快递到哪了？', '退货政策是什么？', '申请退款流程']

// ── 工具函数 ─────────────────────────────────────────────────
function getTime() {
  return new Date().toLocaleTimeString('zh-CN', { hour: '2-digit', minute: '2-digit' })
}

function formatTime(isoTime) {
  if (!isoTime) return ''
  const d = new Date(isoTime)
  return `${d.getMonth()+1}/${d.getDate()} ${d.getHours()}:${String(d.getMinutes()).padStart(2,'0')}`
}

function formatMessage(text) {
  if (!text) return ''
  return text
    .replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;')
    .replace(/\*\*(.*?)\*\*/g, '<strong>$1</strong>')
    .replace(/\n/g, '<br>')
}

async function scrollToBottom() {
  await nextTick()
  if (messagesContainer.value) messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight
}

function autoResize(e) {
  e.target.style.height = 'auto'
  e.target.style.height = Math.min(e.target.scrollHeight, 120) + 'px'
}

function handleEnter(e) {
  if (e.shiftKey) return
  sendMessage()
}

function sendQuick(q) {
  inputText.value = q
  sendMessage()
}

// ── 会话管理 ─────────────────────────────────────────────────

/** 加载用户会话列表 */
async function loadSessionList() {
  try {
    const res = await fetch(`${BASE_URL}/api/sessions/${USER_ID}`)
    const body = await res.json()
    sessionList.value = body.data || []
  } catch (e) {
    console.error('加载会话列表失败', e)
  }
}

/** 新建会话 */
async function createNewSession() {
  try {
    const res = await fetch(`${BASE_URL}/api/sessions?userId=${USER_ID}`, { method: 'POST' })
    const body = await res.json()
    const newSession = body.data
    sessionList.value.unshift(newSession)
    switchSession(newSession)
  } catch (e) {
    console.error('创建会话失败', e)
  }
}

/** 切换会话：从后端加载该会话的历史消息恢复页面 */
async function switchSession(session) {
  if (isStreaming.value) return
  sessionId.value = session.sessionId

  try {
    const res = await fetch(`${BASE_URL}/api/sessions/${session.sessionId}/messages`)
    const body = await res.json()
    messages.value = (body.data || []).map(m => {
      if (m.msgType === 'card') {
        try {
          const payload = JSON.parse(m.content)
          return {
            role: m.role,
            type: 'card',
            cardType: m.cardType,            // 后端持久化的 cardType，直接用，不从 payload 猜
            payload,
            msgId: m.msgId || '',
            submitted: m.submitted === true, // 后端持久化的提交状态
            time: formatTime(m.time),
          }
        } catch (_) { /* 解析失败降级为文本 */ }
      }
      return { role: m.role, content: m.content, time: formatTime(m.time) }
    })
  } catch (e) {
    messages.value = []
  }
  await scrollToBottom()
}

/** 删除会话 */
async function deleteSession(sid) {
  await fetch(`${BASE_URL}/api/sessions/${sid}?userId=${USER_ID}`, { method: 'DELETE' })
  sessionList.value = sessionList.value.filter(s => s.sessionId !== sid)
  if (sessionId.value === sid) {
    if (sessionList.value.length > 0) {
      switchSession(sessionList.value[0])
    } else {
      createNewSession()
    }
  }
}

// ── 发送消息 + 处理 SSE ──────────────────────────────────────
async function sendMessage() {
  const text = inputText.value.trim()
  if (!text || isStreaming.value) return

  messages.value.push({ role: 'user', content: text, time: getTime() })
  inputText.value = ''
  if (inputRef.value) inputRef.value.style.height = 'auto'

  isStreaming.value = true
  streamingBuffer.value = ''
  currentToolStatus.value = ''
  await scrollToBottom()

  try {
    const response = await fetch(`${BASE_URL}/api/chat/stream`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        message: text,
        session_id: sessionId.value,
        userId: USER_ID,
        user_info: userLocation.value ? { lat: userLocation.value.lat, lng: userLocation.value.lng } : {},
      }),
    })

    // 后端在流开始前就报错（4xx/5xx），直接兜底
    if (!response.ok) {
      showError()
      return
    }

    const reader = response.body.getReader()
    currentReader = reader
    const decoder = new TextDecoder()
    let buffer = ''

    while (true) {
      const { done, value } = await reader.read()
      console.log('Received chunk:', { done, value })
      if (done) break
      buffer += decoder.decode(value, { stream: true })
      const lines = buffer.split('\n')
      buffer = lines.pop() || ''
      for (const line of lines) {
        console.log('Processing line:', line)
        if (!line.startsWith('data:')) continue
        const jsonStr = line.slice(5).trim()
        if (!jsonStr) continue
        try { await handleSseEvent(JSON.parse(jsonStr)) } catch (e) {}
      }
    }
  } catch (err) {
    if (err.name !== 'AbortError') showError()
  } finally {
    isStreaming.value = false
    currentToolStatus.value = ''
    currentReader = null
    // 刷新会话列表（标题/时间更新）
    loadSessionList()
  }
}

async function handleSseEvent(event) {
  switch (event.type) {
    case 'token':
      streamingBuffer.value += event.message
      await scrollToBottom()
      break
    case 'tool_start':
      currentToolStatus.value = event.message || `调用 ${event.tool}...`
      await scrollToBottom()
      break
    case 'tool_end':
      currentToolStatus.value = ''
      break
    case 'card':
      if (streamingBuffer.value) {
        messages.value.push({ role: 'assistant', content: streamingBuffer.value, time: getTime() })
        streamingBuffer.value = ''
      }
      messages.value.push({
        role: 'assistant',
        type: 'card',
        cardType: event.card_type,
        payload: event.payload,
        msgId: event.msg_id || '',   // 后端注入的消息 ID，提交时带回
        submitted: false,
        time: getTime(),
      })
      await scrollToBottom()
      break
    case 'error':
      showError(event.message)
      break
    case 'done':
      if (streamingBuffer.value) {
        finalizeStreaming(streamingBuffer.value)
      } else {
        resetStreaming()
      }
      break
    default:
      break
  }
}

function resetStreaming() {
  streamingBuffer.value = ''
  currentToolStatus.value = ''
  isStreaming.value = false
}

function finalizeStreaming(content) {
  if (content) messages.value.push({ role: 'assistant', content, time: getTime() })
  resetStreaming()
  scrollToBottom()
}

function showError(msg) {
  const content = msg || streamingBuffer.value.trim() || DEFAULT_ERROR_MSG
  finalizeStreaming(content)
}

function onCardSubmitted(msg, { message }) {
  msg.submitted = true
  if (message) {
    messages.value.push({ role: 'assistant', content: message, time: getTime() })
    scrollToBottom()
  }
  loadSessionList()
}

function stopStreaming() {
  if (currentReader) {
    currentReader.cancel()
    finalizeStreaming(streamingBuffer.value || '')
  }
}

// ── 初始化 ───────────────────────────────────────────────────
onMounted(async () => {
  requestUserLocation()
  await loadSessionList()
  if (sessionList.value.length > 0) {
    switchSession(sessionList.value[0])
  } else {
    createNewSession()
  }
  inputRef.value?.focus()
})

onUnmounted(() => {
  stopWatchLocation()
})
</script>

<style scoped>
* { box-sizing: border-box; }

.app-layout {
  display: flex;
  height: 100vh;
  font-family: -apple-system, BlinkMacSystemFont, 'PingFang SC', sans-serif;
  background: #f5f7fa;
}

/* 侧边栏 */
.sidebar {
  width: 240px;
  flex-shrink: 0;
  background: #1e1e2e;
  display: flex;
  flex-direction: column;
  border-right: 1px solid #2d2d3f;
}
.sidebar-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 12px;
  border-bottom: 1px solid #2d2d3f;
}
.sidebar-title { color: #aaa; font-size: 13px; }
.new-chat-btn {
  background: #1677ff;
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 5px 10px;
  font-size: 12px;
  cursor: pointer;
  white-space: nowrap;
}
.new-chat-btn:hover { background: #0958d9; }

.session-list { flex: 1; overflow-y: auto; padding: 8px 0; }
.session-item {
  position: relative;
  padding: 10px 14px;
  cursor: pointer;
  border-radius: 8px;
  margin: 2px 8px;
  transition: background .15s;
}
.session-item:hover { background: #2d2d3f; }
.session-item.active { background: #2d2d3f; border-left: 3px solid #1677ff; }
.session-title { color: #ddd; font-size: 13px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; padding-right: 20px; }
.session-time { color: #666; font-size: 11px; margin-top: 3px; }
.del-btn {
  position: absolute; right: 10px; top: 50%; transform: translateY(-50%);
  background: none; border: none; color: #555; cursor: pointer; font-size: 12px;
  opacity: 0;
}
.session-item:hover .del-btn { opacity: 1; }
.del-btn:hover { color: #ff4d4f; }
.no-session { color: #555; font-size: 12px; text-align: center; padding: 20px; }

/* 聊天区域 */
.chat-app {
  flex: 1;
  display: flex;
  flex-direction: column;
  max-width: 100%;
  overflow: hidden;
}

.chat-header {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 14px 20px;
  background: #fff;
  border-bottom: 1px solid #e8e8e8;
  box-shadow: 0 1px 4px rgba(0,0,0,.06);
}
.avatar { font-size: 32px; }
.header-info { flex: 1; }
.name { font-size: 16px; font-weight: 600; color: #1a1a1a; }
.status { font-size: 12px; color: #999; }
.status.online { color: #52c41a; }

.chat-messages {
  flex: 1; overflow-y: auto; padding: 20px 16px;
  display: flex; flex-direction: column; gap: 16px;
}

.welcome { text-align: center; margin: auto; padding: 40px 20px; }
.welcome-icon { font-size: 48px; margin-bottom: 12px; }
.welcome-text { font-size: 18px; font-weight: 600; color: #333; margin-bottom: 6px; }
.welcome-sub { font-size: 13px; color: #999; margin-bottom: 20px; }
.quick-actions { display: flex; flex-wrap: wrap; gap: 8px; justify-content: center; }
.quick-actions button {
  padding: 8px 16px; background: #fff; border: 1px solid #d0d5dd;
  border-radius: 20px; font-size: 13px; color: #555; cursor: pointer; transition: all .2s;
}
.quick-actions button:hover { background: #1677ff; color: #fff; border-color: #1677ff; }

.message { display: flex; align-items: flex-end; gap: 8px; max-width: 85%; }
.message.user { align-self: flex-end; flex-direction: row-reverse; }
.message.assistant { align-self: flex-start; }
.msg-avatar { font-size: 26px; flex-shrink: 0; }
.msg-content { display: flex; flex-direction: column; gap: 4px; }
.message.user .msg-content { align-items: flex-end; }

.tool-status {
  display: inline-flex; align-items: center; gap: 6px;
  font-size: 12px; color: #1677ff; background: #e6f4ff;
  border-radius: 12px; padding: 4px 10px; margin-bottom: 4px;
}
.tool-spinner { display: inline-block; }
.tool-spinner.spinning { animation: spin 1s linear infinite; }
@keyframes spin { from { transform: rotate(0deg); } to { transform: rotate(360deg); } }

.msg-text {
  padding: 10px 14px; border-radius: 16px; font-size: 14px;
  line-height: 1.6; word-break: break-word; max-width: 100%;
}
.message.assistant .msg-text {
  background: #fff; color: #333; border-bottom-left-radius: 4px;
  box-shadow: 0 1px 3px rgba(0,0,0,.08);
}
.message.user .msg-text { background: #1677ff; color: #fff; border-bottom-right-radius: 4px; }
.msg-error { background: #fff2f0 !important; border: 1px solid #ffccc7; color: #cf1322 !important; }
.streaming-text { background: #fff; color: #333; border-bottom-left-radius: 4px; box-shadow: 0 1px 3px rgba(0,0,0,.08); }
.cursor { animation: blink .8s step-end infinite; }
@keyframes blink { 50% { opacity: 0; } }
.msg-time { font-size: 11px; color: #bbb; padding: 0 4px; }

.chat-input-area {
  display: flex; gap: 10px; padding: 12px 16px;
  background: #fff; border-top: 1px solid #e8e8e8; align-items: flex-end;
}
textarea {
  flex: 1; resize: none; border: 1px solid #d0d5dd; border-radius: 12px;
  padding: 10px 14px; font-size: 14px; font-family: inherit; outline: none;
  transition: border-color .2s; min-height: 42px; max-height: 120px; line-height: 1.5;
}
textarea:focus { border-color: #1677ff; box-shadow: 0 0 0 2px rgba(22,119,255,.1); }
textarea:disabled { background: #fafafa; }
.send-btn {
  height: 42px; padding: 0 20px; background: #1677ff; color: #fff;
  border: none; border-radius: 12px; font-size: 14px; cursor: pointer;
  white-space: nowrap; transition: background .2s; flex-shrink: 0;
}
.send-btn:hover:not(:disabled) { background: #0958d9; }
.send-btn:disabled { background: #bfdbfe; cursor: not-allowed; }
.input-hint { text-align: center; font-size: 11px; color: #ccc; padding: 4px 0 8px; background: #fff; display: flex; justify-content: center; align-items: center; gap: 8px; flex-wrap: wrap; }
.loc-status { font-size: 11px; padding: 1px 6px; border-radius: 8px; }
.loc-status.locating { color: #1677ff; }
.loc-status.ok { color: #52c41a; cursor: default; }
.loc-status.fail { color: #ff4d4f; }

.chat-messages::-webkit-scrollbar { width: 4px; }
.chat-messages::-webkit-scrollbar-thumb { background: #ddd; border-radius: 4px; }
.session-list::-webkit-scrollbar { width: 3px; }
.session-list::-webkit-scrollbar-thumb { background: #333; border-radius: 3px; }
</style>

