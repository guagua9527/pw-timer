<script setup>
import { ref, computed, watch } from 'vue'

defineEmits(['switchToPc', 'resetPreference'])

const STORAGE_KEY = 'playtime_records'

const defaultPrice = ref(0)
const balance = ref(0)
const clearBalance = ref(false)
const records = ref([])
const activeIndex = ref(null)
const deductHours = ref(0)
const deductMinutes = ref(0)
const deductPrice = ref(0)
const showSettings = ref(false)

const loadData = () => {
  try {
    const saved = localStorage.getItem(STORAGE_KEY)
    if (saved) {
      const data = JSON.parse(saved)
      defaultPrice.value = data.defaultPrice || 0
      balance.value = data.balance || 0
      records.value = (data.records || []).map(r => {
        if (r.hours > 1000) {
          r.hours = r.hours / 3600000
        }
        if (r.start && typeof r.start === 'number' && r.start > 1000000000000) {
          r.startHour = new Date(r.start).getHours()
          r.startMinute = new Date(r.start).getMinutes()
        }
        if (r.end && typeof r.end === 'number' && r.end > 1000000000000) {
          r.endHour = new Date(r.end).getHours()
          r.endMinute = new Date(r.end).getMinutes()
        }
        return r
      })
      deductHours.value = data.deductHours || 0
      deductMinutes.value = data.deductMinutes || 0
      deductPrice.value = data.deductPrice || 0
    }
  } catch (e) {
    console.error('Failed to load data:', e)
  }
}

const saveData = () => {
  const data = {
    defaultPrice: defaultPrice.value,
    balance: balance.value,
    records: records.value,
    deductHours: deductHours.value,
    deductMinutes: deductMinutes.value,
    deductPrice: deductPrice.value
  }
  localStorage.setItem(STORAGE_KEY, JSON.stringify(data))
}

watch([defaultPrice, balance, records, deductHours, deductMinutes], () => {
  if (deductPrice.value === 0) {
    deductPrice.value = defaultPrice.value
  }
  saveData()
}, { deep: true })

const isTiming = computed(() => activeIndex.value !== null)

const totalPrice = computed(() => {
  return records.value.reduce((sum, r) => sum + (r.total || 0), 0)
})

const totalHours = computed(() => {
  const recordHours = records.value.reduce((sum, r) => sum + (r.hours || 0), 0)
  const deductTotalMinutes = (deductHours.value || 0) * 60 + (deductMinutes.value || 0)
  return Math.max(0, recordHours - deductTotalMinutes / 60)
})

const deductAmount = computed(() => {
  return ((deductHours.value || 0) * 60 + (deductMinutes.value || 0)) * (deductPrice.value || 0) / 60
})

const consumption = computed(() => {
  return Math.max(0, totalPrice.value - deductAmount.value)
})

const remaining = computed(() => {
  return balance.value - consumption.value
})

const reversedRecords = computed(() => {
  return [...records.value].reverse()
})

const formatDuration = (hours) => {
  const totalMinutes = Math.round(hours * 60)
  const h = Math.floor(totalMinutes / 60)
  const m = totalMinutes % 60
  return `${h}小时${m}分`
}

const formatTime = (hour, minute) => {
  if (hour === null || minute === null) return '--:--'
  return `${String(hour).padStart(2, '0')}:${String(minute).padStart(2, '0')}`
}

const editingIndex = ref(null)
const editingField = ref('')
const editHour = ref(0)
const editMinute = ref(0)

const hourOptions = Array.from({ length: 24 }, (_, i) => ({
  label: i.toString().padStart(2, '0'),
  value: i
}))

const minuteOptions = Array.from({ length: 60 }, (_, i) => ({
  label: i.toString().padStart(2, '0'),
  value: i
}))

const startEdit = (realIndex, field) => {
  const originalIndex = records.value.length - 1 - realIndex
  const record = records.value[originalIndex]
  editingIndex.value = originalIndex
  editingField.value = field
  const hour = field === 'start' ? record.startHour : record.endHour
  const minute = field === 'start' ? record.startMinute : record.endMinute
  editHour.value = hour !== null ? hour : new Date().getHours()
  editMinute.value = minute !== null ? minute : new Date().getMinutes()
}

const handleTimeChange = () => {
  if (editingIndex.value === null) return
  const record = records.value[editingIndex.value]
  if (editingField.value === 'start') {
    record.startHour = editHour.value
    record.startMinute = editMinute.value
  } else {
    record.endHour = editHour.value
    record.endMinute = editMinute.value
  }
  recalcRecord(editingIndex.value)
  cancelEdit()
}

const cancelEdit = () => {
  editingIndex.value = null
  editingField.value = ''
}

const isEditing = (realIndex, field) => {
  const originalIndex = records.value.length - 1 - realIndex
  return editingIndex.value === originalIndex && editingField.value === field
}

const calcDuration = (startHour, startMinute, endHour, endMinute) => {
  let durationMinutes = endHour * 60 + endMinute - (startHour * 60 + startMinute)
  if (durationMinutes < 0) {
    durationMinutes += 24 * 60
  }
  return durationMinutes
}

const handleStartTimer = () => {
  const now = new Date()
  records.value.push({
    startHour: now.getHours(),
    startMinute: now.getMinutes(),
    endHour: null,
    endMinute: null,
    price: defaultPrice.value,
    hours: 0,
    total: 0
  })
  activeIndex.value = records.value.length - 1
}

const handleEndTimer = () => {
  if (activeIndex.value === null) return
  const record = records.value[activeIndex.value]
  const now = new Date()
  record.endHour = now.getHours()
  record.endMinute = now.getMinutes()
  const minutes = calcDuration(record.startHour, record.startMinute, record.endHour, record.endMinute)
  record.hours = minutes / 60
  record.total = parseFloat((record.hours * record.price).toFixed(2))
  activeIndex.value = null
}

const handleDelete = (index) => {
  records.value.splice(index, 1)
  if (activeIndex.value === index) {
    activeIndex.value = null
  } else if (activeIndex.value > index) {
    activeIndex.value--
  }
}

const handleAddRecord = () => {
  records.value.push({
    startHour: null,
    startMinute: null,
    endHour: null,
    endMinute: null,
    price: defaultPrice.value,
    hours: 0,
    total: 0
  })
}

const handleClear = () => {
  records.value = []
  activeIndex.value = null
  deductHours.value = 0
  deductMinutes.value = 0
  deductPrice.value = 0
  if (clearBalance.value) {
    balance.value = 0
    clearBalance.value = false
  }
}

const handleDeduct = () => {
  balance.value = balance.value - consumption.value
  records.value = []
  activeIndex.value = null
}

const recalcRecord = (index) => {
  const record = records.value[index]
  if (record.startHour !== null && record.startMinute !== null &&
      record.endHour !== null && record.endMinute !== null) {
    const minutes = calcDuration(record.startHour, record.startMinute, record.endHour, record.endMinute)
    record.hours = Math.round(minutes / 60 * 100) / 100
    record.total = Math.round(record.hours * record.price * 100) / 100
  } else {
    record.hours = 0
    record.total = 0
  }
}

loadData()
</script>

<template>
  <div class="mobile-app">
      <!-- Header -->
      <header class="mobile-header">
        <span class="header-icon">🕐</span>
        <span class="header-title">瓜瓜的陪玩计时工具</span>
        <div class="header-actions">
          <button class="view-toggle-btn" @click="$emit('switchToPc')">电脑版</button>
          <button class="settings-btn" @click="showSettings = !showSettings">
            {{ showSettings ? '✕' : '⚙' }}
          </button>
        </div>
      </header>

      <!-- Timer Button -->
      <div class="timer-section">
        <button
          class="timer-btn"
          :class="{ timing: isTiming }"
          @click="isTiming ? handleEndTimer() : handleStartTimer()"
        >
          {{ isTiming ? '⏸ 结束计时' : '🎯 开始计时' }}
        </button>
      </div>

      <!-- Settings Panel -->
      <div v-if="showSettings" class="settings-panel">
        <div class="setting-item">
          <label>默认单价</label>
          <n-input-number
            v-model:value="defaultPrice"
            :min="0"
            :precision="2"
            size="small"
            style="width: 120px;"
          >
            <template #suffix>¥/h</template>
          </n-input-number>
        </div>
        <div class="setting-item">
          <label>存单余额</label>
          <n-input-number
            v-model:value="balance"
            :precision="2"
            size="small"
            style="width: 120px;"
          >
            <template #suffix>¥</template>
          </n-input-number>
        </div>
        <div class="setting-item">
          <label>扣减时长</label>
          <n-input-number v-model:value="deductHours" :min="0" :precision="0" size="small" style="width: 60px;" />
          <span>时</span>
          <n-input-number v-model:value="deductMinutes" :min="0" :precision="0" size="small" style="width: 60px;" />
          <span>分</span>
        </div>
        <div class="setting-item">
          <label>扣减单价</label>
          <n-input-number v-model:value="deductPrice" :min="0" :precision="0" size="small" style="width: 80px;" />
          <span>¥/h</span>
        </div>
        <div class="setting-item">
          <button class="reset-btn" @click="$emit('resetPreference')">重置设备偏好</button>
        </div>
      </div>

      <!-- Records List -->
      <div class="records-section">
        <div class="section-header">
          <span class="section-title">计时记录</span>
          <button class="add-btn" @click="handleAddRecord">+ 添加</button>
        </div>

        <!-- Empty State -->
        <div v-if="records.length === 0" class="empty-state">
          暂无记录，点击上方按钮开始计时
        </div>

        <!-- Record Cards -->
        <div v-else class="records-list">
          <div v-for="(record, realIndex) in reversedRecords" :key="realIndex" class="record-card">
            <div class="card-header">
              <span class="record-index">#{{ records.length - realIndex }}</span>
              <button class="delete-btn" @click="handleDelete(records.length - 1 - realIndex)">删除</button>
            </div>
            <div class="card-content">
              <div class="card-row">
                <span class="card-label">开始</span>
                <div v-if="isEditing(realIndex, 'start')" class="time-edit">
                  <select v-model="editHour" @change="handleTimeChange" class="time-select">
                    <option v-for="h in hourOptions" :key="h.value" :value="h.value">{{ h.label }}</option>
                  </select>
                  <span>:</span>
                  <select v-model="editMinute" @change="handleTimeChange" class="time-select">
                    <option v-for="m in minuteOptions" :key="m.value" :value="m.value">{{ m.label }}</option>
                  </select>
                </div>
                <span v-else class="card-value time" @click="startEdit(realIndex, 'start')">{{ formatTime(record.startHour, record.startMinute) }}</span>
              </div>
              <div class="card-row">
                <span class="card-label">结束</span>
                <div v-if="isEditing(realIndex, 'end')" class="time-edit">
                  <select v-model="editHour" @change="handleTimeChange" class="time-select">
                    <option v-for="h in hourOptions" :key="h.value" :value="h.value">{{ h.label }}</option>
                  </select>
                  <span>:</span>
                  <select v-model="editMinute" @change="handleTimeChange" class="time-select">
                    <option v-for="m in minuteOptions" :key="m.value" :value="m.value">{{ m.label }}</option>
                  </select>
                </div>
                <span v-else class="card-value time" @click="startEdit(realIndex, 'end')">{{ formatTime(record.endHour, record.endMinute) }}</span>
              </div>
              <div class="card-divider"></div>
              <div class="card-row">
                <span class="card-label">时长</span>
                <span class="card-value highlight">{{ formatDuration(record.hours) }}</span>
              </div>
              <div class="card-row">
                <span class="card-label">单价</span>
                <span class="card-value">{{ record.price }} ¥/h</span>
              </div>
              <div class="card-row total">
                <span class="card-label">总价</span>
                <span class="card-value price">¥{{ record.total?.toFixed(2) || '0.00' }}</span>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- Summary -->
      <div v-if="records.length > 0" class="summary-section">
        <div class="summary-row">
          <span>合计时长</span>
          <span class="summary-value">{{ formatDuration(totalHours) }}</span>
        </div>
        <div class="summary-row">
          <span>消费金额</span>
          <span class="summary-value">¥ {{ consumption.toFixed(2) }}</span>
        </div>
        <div class="summary-row" v-if="deductAmount > 0">
          <span>扣减金额</span>
          <span class="summary-value deduct">-¥ {{ deductAmount.toFixed(2) }}</span>
        </div>
        <div class="summary-row">
          <span>剩余金额</span>
          <span class="summary-value" :class="{ negative: remaining < 0 }">
            ¥ {{ remaining.toFixed(2) }}
          </span>
        </div>
      </div>

      <!-- Action Buttons -->
      <div class="action-section">
        <button class="action-btn deduct-btn" @click="handleDeduct">
          💸 扣费
        </button>
        <button class="action-btn clear-btn" @click="handleClear">
          🗑 清空
        </button>
      </div>
    </div>
</template>

<script>
import { NInputNumber } from 'naive-ui'

export default {
  components: {
    NInputNumber
  }
}
</script>

<style scoped>
.mobile-app {
  min-height: 100vh;
  background: #FAF7F2;
  padding-bottom: 100px;
}

.mobile-header {
  display: flex;
  align-items: center;
  padding: 12px 16px;
  background: #FFFFFF;
  border-bottom: 1px solid #EAE0D5;
  position: sticky;
  top: 0;
  z-index: 100;
}

.header-icon {
  font-size: 24px;
  margin-right: 8px;
}

.header-title {
  flex: 1;
  font-size: 18px;
  font-weight: bold;
  color: #3D3D3D;
}

.header-actions {
  display: flex;
  gap: 8px;
}

.view-toggle-btn {
  padding: 6px 12px;
  border: 1px solid #EAE0D5;
  background: #FFFFFF;
  border-radius: 8px;
  font-size: 12px;
  color: #3D3D3D;
  cursor: pointer;
}

.reset-btn {
  width: 100%;
  padding: 8px;
  border: 1px solid #EAE0D5;
  background: #FFFFFF;
  border-radius: 6px;
  font-size: 14px;
  color: #8A8A8A;
  cursor: pointer;
}

.settings-btn {
  width: 36px;
  height: 36px;
  border: none;
  background: #F5F5F5;
  border-radius: 8px;
  font-size: 18px;
  cursor: pointer;
}

.timer-section {
  padding: 20px 16px;
  display: flex;
  justify-content: center;
}

.timer-btn {
  width: 100%;
  max-width: 300px;
  height: 56px;
  border: none;
  border-radius: 12px;
  background: #E8985E;
  color: #FFFFFF;
  font-size: 18px;
  font-weight: bold;
  box-shadow: 0 4px 12px rgba(0,0,0,0.15);
  cursor: pointer;
}

.timer-btn.timing {
  background: #7CAE7A;
}

.settings-panel {
  background: #FFFFFF;
  margin: 0 16px 16px;
  padding: 16px;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.06);
}

.setting-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 12px;
  gap: 12px;
}

.setting-item:last-child {
  margin-bottom: 0;
}

.setting-item label {
  font-size: 14px;
  color: #3D3D3D;
  min-width: 70px;
}

.setting-item span {
  font-size: 14px;
  color: #8A8A8A;
  margin-left: 4px;
}

.records-section {
  padding: 0 16px;
}

.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}

.section-title {
  font-size: 16px;
  font-weight: 500;
  color: #3D3D3D;
}

.add-btn {
  padding: 6px 16px;
  border: 1px solid #E8985E;
  background: transparent;
  color: #E8985E;
  border-radius: 6px;
  font-size: 14px;
  cursor: pointer;
}

.empty-state {
  text-align: center;
  padding: 40px 20px;
  color: #8A8A8A;
  font-size: 14px;
  background: #FFFFFF;
  border-radius: 12px;
}

.records-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.record-card {
  background: #FFFFFF;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.06);
  overflow: hidden;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 14px;
  background: #F5F5F5;
}

.record-index {
  font-size: 13px;
  font-weight: 500;
  color: #8A8A8A;
}

.delete-btn {
  padding: 4px 12px;
  border: none;
  background: #D96E6E;
  color: #FFFFFF;
  border-radius: 4px;
  font-size: 12px;
  cursor: pointer;
}

.card-content {
  padding: 14px;
}

.card-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.card-row:last-child {
  margin-bottom: 0;
}

.card-label {
  font-size: 14px;
  color: #8A8A8A;
}

.card-value {
  font-size: 14px;
  color: #3D3D3D;
}

.card-value.time {
  font-family: monospace;
  font-size: 16px;
}

.card-value.highlight {
  color: #E8985E;
  font-weight: 500;
}

.card-value.price {
  font-size: 18px;
  font-weight: bold;
  color: #E8985E;
}

.card-divider {
  height: 1px;
  background: #EAE0D5;
  margin: 10px 0;
}

.card-row.total {
  margin-top: 4px;
}

.summary-section {
  margin: 20px 16px;
  padding: 16px;
  background: #FFFFFF;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.06);
}

.summary-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 0;
  font-size: 14px;
  color: #3D3D3D;
}

.summary-row.deduct {
  color: #8A8A8A;
}

.summary-value {
  font-weight: 500;
}

.summary-value.negative {
  color: #D96E6E;
}

.summary-value.deduct {
  color: #8A8A8A;
}

.action-section {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  display: flex;
  gap: 12px;
  padding: 16px;
  background: #FFFFFF;
  border-top: 1px solid #EAE0D5;
}

.action-btn {
  flex: 1;
  height: 48px;
  border: none;
  border-radius: 8px;
  font-size: 16px;
  font-weight: 500;
  cursor: pointer;
}

.deduct-btn {
  background: #E8985E;
  color: #FFFFFF;
}

.clear-btn {
  background: #F5F5F5;
  color: #D96E6E;
}

.time-edit {
  display: flex;
  align-items: center;
  gap: 4px;
}

.time-select {
  width: 50px;
  height: 28px;
  border: 1px solid #EAE0D5;
  border-radius: 4px;
  background: #FFFFFF;
  font-size: 14px;
  padding: 0 4px;
  text-align: center;
}

.edit-confirm, .edit-cancel {
  width: 28px;
  height: 28px;
  border: none;
  border-radius: 4px;
  font-size: 14px;
  cursor: pointer;
}

.edit-confirm {
  background: #7CAE7A;
  color: #FFFFFF;
}

.edit-cancel {
  background: #D96E6E;
  color: #FFFFFF;
}

.setting-item .n-input-number {
  flex: 1;
  min-width: 80px;
}

.setting-item .n-input-number input {
  text-align: center;
}
</style>
