<template>
  <div class="card-wrap" :class="{ 'card-danger': template?.danger }">
    <div class="card-title">
      {{ cardType === 'nearby_poi' && payload.keywords
          ? `📍 附近${payload.keywords}`
          : (template?.title || cardType) }}
    </div>

    <!-- 加载中 -->
    <div v-if="loading" class="card-loading">加载中...</div>

    <!-- 已提交只读态 -->
    <div v-else-if="submitted" class="card-submitted">
      <span>✅ {{ submitDoneLabel }}</span>
      <div v-if="confirmMessage" class="card-confirm-msg">{{ confirmMessage }}</div>
      <div v-else class="card-confirm-msg">操作已完成，请等待后续处理。</div>
    </div>

    <!-- POI 展示卡（超充站 / 洗车店 / 加油站 / 停车场等通用） -->
    <div v-else-if="template?.category === 'display' && isPoiCard">
      <div v-if="!poiItems.length" class="card-empty">暂无附近数据</div>
      <div v-for="s in poiItems" :key="s.id" class="station-item">
        <div class="station-name">{{ s.name }}</div>
        <div class="station-addr">📍 {{ s.address }}</div>
        <div v-if="s.distance" class="station-distance">🚶 距您 {{ formatDistance(s.distance) }}</div>
        <!-- 充电站专有字段 -->
        <div v-if="cardType === 'charging_station'" class="station-status">
          <span v-if="s.available !== null && s.available !== undefined"
                :class="s.available > 0 ? 'pile-ok' : 'pile-full'">
            ⚡ 空闲 {{ s.available }}/{{ s.totalPiles }} 桩 · {{ s.maxPower }}kW {{ s.pileType }}
          </span>
          <span v-else class="pile-ok">
            ⚡ {{ s.maxPower }}kW {{ s.pileType }} 超充
          </span>
        </div>
        <div v-if="s.tel" class="station-tel">📞 {{ s.tel }}</div>
        <!-- 营业时间（通用 POI 可能返回）-->
        <div v-if="s.businessHours" class="station-hours">🕐 {{ s.businessHours }}</div>
      </div>
    </div>

    <!-- 表单卡 -->
    <template v-else-if="template?.category === 'form'">
      <div v-for="field in template.fields" :key="field.key" class="card-field">
        <label>{{ field.label }}<span v-if="field.required" class="req">*</span></label>

        <!-- 只读展示 -->
        <div v-if="field.type === 'readonly'" class="field-readonly">
          {{ form[field.key] || '—' }}
        </div>

        <!-- 信息说明（取消政策等） -->
        <div v-else-if="field.type === 'info'" class="field-info">
          {{ form[field.key] }}
        </div>

        <!-- 勾选框 -->
        <label v-else-if="field.type === 'checkbox'" class="field-checkbox">
          <input type="checkbox" v-model="form[field.key]" />
          <span>{{ field.label }}</span>
        </label>

        <!-- 城市选择已由后端移除，前端不再展示 city-select -->

        <!-- 服务中心/体验中心选择 -->
        <select v-else-if="field.type === 'center-select'"
                v-model="form[field.key]"
                :disabled="(cardType !== 'test_drive') && field.dependsOn && !form[field.dependsOn]">
          <option value="" disabled>{{ (cardType === 'test_drive' || !field.dependsOn || form[field.dependsOn]) ? '请选择服务中心' : '请先选择城市' }}</option>
          <option v-for="c in filteredCenters" :key="c.id" :value="c.id">
            {{ c.name }}（{{ c.address }}）
          </option>
        </select>

        <!-- Payload 动态选择（车型等） -->
        <select v-else-if="field.type === 'payload-select'" v-model="form[field.key]">
          <option value="" disabled>请选择</option>
          <option v-for="opt in getPayloadOptions(field)" :key="opt" :value="opt">{{ opt }}</option>
        </select>

        <!-- 静态 select -->
        <select v-else-if="field.type === 'select'" v-model="form[field.key]">
          <option value="" disabled>请选择</option>
          <option v-for="opt in field.options" :key="opt.value" :value="opt.value">{{ opt.label }}</option>
        </select>

        <!-- datetime -->
        <input v-else-if="field.type === 'datetime'" type="datetime-local"
               v-model="form[field.key]" />

        <!-- tel / text -->
        <input v-else :type="field.type === 'tel' ? 'tel' : 'text'"
               v-model="form[field.key]"
               :placeholder="field.placeholder || ''" />
      </div>

      <button class="card-submit-btn" :disabled="!canSubmit || submitting"
              :class="{ 'btn-danger': template?.danger }"
              @click="submit">
        {{ submitting ? '提交中...' : (template?.submitLabel || '确认') }}
      </button>
    </template>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'

const props = defineProps({
  cardType: { type: String, required: true },
  payload: { type: Object, default: () => ({}) },
  sessionId: { type: String, default: '' },
  userId: { type: String, default: '' },
  msgId: { type: String, default: '' },        // 后端消息 ID，提交时带回以标记已提交
  initSubmitted: { type: Boolean, default: false },
})

const emit = defineEmits(['submitted'])

// expose `cardType` for use in script (template already has access)
const cardType = props.cardType

const BASE_URL = 'http://localhost:8080'

const template = ref(null)
const loading = ref(true)
const submitted = ref(props.initSubmitted)
const submitting = ref(false)
const confirmMessage = ref('')
const form = ref({})

// 模版缓存（避免重复请求）
const _cache = {}

const submitDoneLabel = computed(() => {
  const map = {
    appointment: '预约已提交',
    test_drive: '试驾预约已提交',
    refund_confirm: '退款申请已提交',
    cancel_order_confirm: '订单已取消',
  }
  return map[props.cardType] || '操作已完成'
})

const centersAll = computed(() => props.payload?.centers_structured || [])

// POI 展示卡相关
const POI_CARD_TYPES = ['charging_station', 'nearby_poi']
const isPoiCard = computed(() => POI_CARD_TYPES.includes(props.cardType))
// 兼容 payload.stations（超充站旧字段）和 payload.items（通用 POI）
const poiItems = computed(() => props.payload?.stations || props.payload?.items || [])

const filteredCenters = computed(() => {
  // centersAll already contains all centers provided by backend
  return centersAll.value
})

function onCityChange(cityKey) {
  // 重置依赖城市的字段
  const centerField = template.value?.fields?.find(f => f.dependsOn === cityKey)
  if (centerField) form.value[centerField.key] = ''
}

function formatDistance(meters) {
  const m = parseInt(meters)
  if (isNaN(m)) return meters
  return m >= 1000 ? `${(m / 1000).toFixed(1)}km` : `${m}m`
}

function getPayloadOptions(field) {
  const key = field.optionsKey || field.key + 's'
  const opts = props.payload[key]
  if (Array.isArray(opts)) return opts
  return []
}

const canSubmit = computed(() => {
  if (!template.value?.fields) return false
  return template.value.fields.every(f => {
    if (!f.required) return true
    const val = form.value[f.key]
    if (f.type === 'checkbox') return val === true
    return val !== undefined && val !== '' && val !== null
  })
})

async function fetchTemplate() {
  if (_cache[props.cardType]) {
    template.value = _cache[props.cardType]
    loading.value = false
    initForm()
    return
  }
  try {
    const res = await fetch(`${BASE_URL}/api/cards/templates/${props.cardType}`)
    const body = await res.json()
    if (body.code === 200) {
      template.value = body.data
      _cache[props.cardType] = body.data
    }
  } catch (e) {
    console.error('加载卡片模版失败', e)
  } finally {
    loading.value = false
    initForm()
  }
}

function initForm() {
  if (!template.value?.fields) return
  const prefill = props.payload?.prefill || {}
  const f = {}
  for (const field of template.value.fields) {
    if (field.type === 'checkbox') {
      f[field.key] = false
    } else if (field.prefillKey && prefill[field.prefillKey] != null) {
      f[field.key] = String(prefill[field.prefillKey])
    } else {
      f[field.key] = ''
    }
  }
  form.value = f
}

async function submit() {
  if (!canSubmit.value || submitting.value) return
  submitting.value = true
  try {
    const body = {
      sessionId: props.sessionId,
      userId: props.userId,
      msgId: props.msgId,   // 后端用于标记该卡片已提交
      ...form.value,
      ...(props.payload.issue_description ? { issueDescription: props.payload.issue_description } : {}),
    }
    // 勾选框不需要传后端
    for (const field of (template.value?.fields || [])) {
      if (field.type === 'checkbox') delete body[field.key]
    }

    // 若存在 center-select 字段，尝试把所选中心的完整信息附加到 body，便于后端匹配
    try {
      const centerField = template.value?.fields?.find(f => f.type === 'center-select')
      if (centerField && body[centerField.key]) {
        const selectedId = body[centerField.key]
        const centerObj = centersAll.value.find(c => c.id === selectedId)
        if (centerObj) {
          body.selectedCenter = centerObj
        } else {
          console.warn('[CardRenderer] 提交时未在 payload 中找到所选中心详情，id=', selectedId)
        }
      }
    } catch (e) {
      console.warn('[CardRenderer] 附加中心信息失败', e.message)
    }

    console.log('[CardRenderer] 提交 body]', body)

    const res = await fetch(`${BASE_URL}${template.value.submitApi}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
    })
    const result = await res.json()
    if (res.ok && result.code === 200) {
      submitted.value = true
      confirmMessage.value = result.data?.message || '操作成功'
      emit('submitted', { cardType: props.cardType, message: confirmMessage.value })
    } else {
      alert(result.message || '提交失败，请稍后重试')
    }
  } catch (e) {
    alert('网络异常，请稍后重试')
  } finally {
    submitting.value = false
  }
}

onMounted(fetchTemplate)
</script>

<style scoped>
.card-wrap {
  background: #fff;
  border: 1px solid #e0e7ff;
  border-radius: 12px;
  padding: 16px;
  min-width: 300px;
  max-width: 400px;
  box-shadow: 0 2px 8px rgba(22,119,255,.1);
}
.card-wrap.card-danger { border-color: #ffa39e; box-shadow: 0 2px 8px rgba(255,77,79,.1); }
.card-title { font-size: 15px; font-weight: 600; color: #1677ff; margin-bottom: 14px; }
.card-wrap.card-danger .card-title { color: #cf1322; }
.card-loading { color: #999; font-size: 13px; }
.card-empty { color: #999; font-size: 13px; padding: 8px 0; }

/* 超充站展示 */
.station-item { border-bottom: 1px solid #f0f0f0; padding: 8px 0; }
.station-item:last-child { border-bottom: none; }
.station-name { font-size: 13px; font-weight: 500; color: #333; }
.station-addr { font-size: 12px; color: #888; margin: 2px 0; }
.station-distance { font-size: 12px; color: #4CAF50; margin: 2px 0; }
.station-tel { font-size: 12px; color: #888; margin: 2px 0; }
.station-hours { font-size: 12px; color: #888; margin: 2px 0; }
.station-status { font-size: 12px; }
.pile-ok { color: #52c41a; }
.pile-full { color: #ff4d4f; }

/* 表单 */
.card-field { display: flex; flex-direction: column; gap: 4px; margin-bottom: 12px; }
.card-field label { font-size: 12px; color: #888; }
.req { color: #ff4d4f; margin-left: 2px; }
.card-field select, .card-field input[type="text"],
.card-field input[type="tel"], .card-field input[type="datetime-local"] {
  border: 1px solid #d0d5dd; border-radius: 8px;
  padding: 8px 10px; font-size: 13px; outline: none;
  transition: border-color .2s;
}
.card-field select:focus, .card-field input:focus { border-color: #1677ff; }
.card-field select:disabled { background: #fafafa; color: #bbb; }
.field-readonly { padding: 8px 10px; background: #f5f5f5; border-radius: 8px; font-size: 13px; color: #555; }
.field-info { padding: 8px 10px; background: #fffbe6; border: 1px solid #ffe58f; border-radius: 8px; font-size: 12px; color: #875800; line-height: 1.5; }
.field-checkbox { display: flex; align-items: flex-start; gap: 8px; font-size: 13px; color: #333; cursor: pointer; }
.field-checkbox input { margin-top: 2px; flex-shrink: 0; }

.card-submit-btn {
  width: 100%; padding: 10px; background: #1677ff; color: #fff;
  border: none; border-radius: 8px; font-size: 14px; cursor: pointer;
  margin-top: 4px; transition: background .2s;
}
.card-submit-btn:hover:not(:disabled) { background: #0958d9; }
.card-submit-btn:disabled { background: #bfdbfe; cursor: not-allowed; }
.card-submit-btn.btn-danger { background: #ff4d4f; }
.card-submit-btn.btn-danger:hover:not(:disabled) { background: #d9363e; }

.card-submitted { display: flex; flex-direction: column; gap: 6px; color: #52c41a; font-size: 14px; font-weight: 500; }
.card-confirm-msg { font-size: 12px; color: #666; font-weight: 400; white-space: pre-line; }
</style>


