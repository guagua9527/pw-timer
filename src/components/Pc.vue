<script setup>
import { ref, computed, watch } from 'vue'

defineEmits(['switchToMobile'])

const STORAGE_KEY = 'playtime_records'

const defaultPrice = ref(0)
const balance = ref(0)
const clearBalance = ref(false)
const records = ref([])
const activeIndex = ref(null)
const deductHours = ref(0)
const deductMinutes = ref(0)
const deductPrice = ref(0)

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

const formatDuration = (hours) => {
  const totalMinutes = Math.round(hours * 60)
  const h = Math.floor(totalMinutes / 60)
  const m = totalMinutes % 60
  return `${h}小时${m}分`
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

const updateRecordHour = (index, field, hour) => {
  records.value[index][field + 'Hour'] = hour
  recalcRecord(index)
}

const updateRecordMinute = (index, field, minute) => {
  records.value[index][field + 'Minute'] = minute
  recalcRecord(index)
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

const updateRecordPrice = (index, value) => {
  records.value[index].price = value || 0
  records.value[index].total = Math.round(records.value[index].hours * records.value[index].price * 100) / 100
}

loadData()

const columns = [
  {
    title: '开始时间',
    key: 'start',
    width: 100,
    align: 'center',
    render: (row, index) => {
      return h(TimePicker, {
        hour: row.startHour,
        minute: row.startMinute,
        now: row.startHour === null,
        placeholder: '输入',
        'onUpdate:hour': (val) => updateRecordHour(index, 'start', val),
        'onUpdate:minute': (val) => updateRecordMinute(index, 'start', val),
        style: { width: '100%' }
      })
    }
  },
  {
    title: '结束时间',
    key: 'end',
    width: 100,
    align: 'center',
    render: (row, index) => {
      return h(TimePicker, {
        hour: row.endHour,
        minute: row.endMinute,
        now: row.endHour === null,
        placeholder: '输入',
        'onUpdate:hour': (val) => updateRecordHour(index, 'end', val),
        'onUpdate:minute': (val) => updateRecordMinute(index, 'end', val),
        style: { width: '100%' }
      })
    }
  },
  {
    title: '单价(¥/h)',
    key: 'price',
    width: 120,
    align: 'center',
    render: (row, index) => {
      return h(NInputNumber, {
        value: row.price,
        min: 0,
        precision: 2,
        showButton: false,
        onUpdateValue: (val) => updateRecordPrice(index, val),
        style: { width: '100%' }
      })
    }
  },
  {
    title: '时长',
    key: 'hours',
    width: 120,
    align: 'center',
    render: (row) => {
      return formatDuration(row.hours)
    }
  },
  {
    title: '总价(¥)',
    key: 'total',
    width: 120,
    align: 'center',
    render: (row) => {
      return row.total?.toFixed(2) || '0.00'
    }
  },
  {
    title: '操作',
    key: 'actions',
    width: 80,
    align: 'center',
    render: (row, index) => {
      return h(NButton, {
        type: 'error',
        size: 'small',
        onClick: () => handleDelete(index)
      }, { default: () => '删除' })
    }
  }
]
</script>

<template>
  <n-layout style="min-height: 100vh; background: #FAF7F2;">
    <n-layout-header style="padding: 16px 24px; background: #FFFFFF; border-bottom: 1px solid #EAE0D5; display: flex; justify-content: center; align-items: center;">
      <n-space align="center" justify="center" :size="12">
        <span style="font-size: 24px;">🕐</span>
        <n-text strong style="font-size: 20px; color: #3D3D3D;">瓜瓜的陪玩计时工具</n-text>
      </n-space>
      <n-button size="small" style="position: absolute; right: 24px;" @click="$emit('switchToMobile')">手机版</n-button>
    </n-layout-header>

    <n-layout-content style="padding: 24px;">
      <div style="display: flex; gap: 24px; align-items: flex-start;">
        <div style="flex: 1;">
          <n-card :bordered="false" style="background: #FFFFFF; border-radius: 12px; box-shadow: 0 2px 12px rgba(0,0,0,0.06); padding: 24px;">
            <n-space vertical :size="16" style="width: 100%;">
              <n-space vertical :size="12" align="center">
                <n-button
                  type="default"
                  :style="{
                    background: isTiming ? '#7CAE7A' : '#E8985E',
                    color: '#FFFFFF',
                    border: 'none',
                    borderRadius: '12px',
                    height: '56px',
                    fontSize: '18px',
                    fontWeight: 'bold',
                    padding: '0 48px',
                    boxShadow: '0 4px 12px rgba(0,0,0,0.15)'
                  }"
                  @click="isTiming ? handleEndTimer() : handleStartTimer()"
                >
                  {{ isTiming ? '⏸ 结束计时' : '🎯 开始计时' }}
                </n-button>
              </n-space>

              <n-data-table
                :columns="columns"
                :data="records"
                :bordered="true"
                :single-line="false"
                style="text-align: center;"
              />
            </n-space>

            <n-space vertical :size="8" style="justify-content: flex-end;">
                <n-button
                  type="default"
                  block
                  @click="handleAddRecord"
                >
                  + 添加记录
                </n-button>

                <n-space v-if="records.length > 0" align="center" justify="flex-end" :size="8" style="padding: 12px 16px; background: #F5F5F5; border-radius: 8px;">
                  <n-text style="color: #3D3D3D; font-size: 14px;">扣减单价:</n-text>
                  <n-input-number
                    v-model:value="deductPrice"
                    :min="0"
                    :precision="0"
                    size="small"
                    button-placement="both"
                    style="width: 100px; text-align: center;"
                  />
                  <n-text style="color: #3D3D3D; font-size: 14px;">¥/h</n-text>
                  <n-text style="color: #3D3D3D; font-size: 14px;">扣减时长:</n-text>
                  <n-input-number
                    v-model:value="deductHours"
                    :min="0"
                    :precision="0"
                    size="small"
                    button-placement="both"
                    style="width: 80px; text-align: center;"
                  />
                  <n-text style="color: #3D3D3D; font-size: 14px;">小时</n-text>
                  <n-input-number
                    v-model:value="deductMinutes"
                    :min="0"
                    :precision="0"
                    size="small"
                    button-placement="both"
                    style="width: 100px; text-align: center;"
                  />
                  <n-text style="color: #3D3D3D; font-size: 14px;">分钟</n-text>

                  <n-text style="color: #3D3D3D; font-size: 14px;">扣减:</n-text>
                  <n-text strong style="color: #E8985E; font-size: 18px;">¥{{ deductAmount.toFixed(2) }}</n-text>

                </n-space>
                <n-space v-if="records.length > 0" :size="24" align="center" justify="flex-end" style="font-size: 24px; padding: 12px 16px; background: #F5F5F5; border-radius: 8px; min-width: 300px;">
                  <n-text strong style="color: #3D3D3D;">合计:</n-text>
                  <n-text strong style="color: #3D3D3D;">{{ formatDuration(totalHours) }}</n-text>
                  <n-text strong style="color: #E8985E;">¥{{ consumption.toFixed(2) }}</n-text>
                </n-space>
              </n-space>
          </n-card>
        </div>

        <div style="width: 30%; min-width: 320px;">
          <n-card :bordered="false" style="background: #FFFFFF; border-radius: 12px; box-shadow: 0 2px 12px rgba(0,0,0,0.06); padding: 24px;">
            <n-space vertical :size="20">
              <n-space vertical :size="12">
                <n-text strong style="color: #3D3D3D; font-size: 16px;">💱 默认单价</n-text>
                <n-input-number
                  v-model:value="defaultPrice"
                  :min="0"
                  :precision="2"
                  show-button
                  placeholder="¥/h"
                  :show-edge="false"
                  size="large"
                  style="width: 100%; font-size: 18px;"
                >
                  <template #suffix>¥/h</template>
                </n-input-number>
              </n-space>

              <n-space vertical :size="12">
                <n-text strong style="color: #3D3D3D; font-size: 16px;">💰 存单余额</n-text>
                <n-input-number
                  v-model:value="balance"
                  :precision="2"
                  show-button
                  placeholder="¥"
                  size="large"
                  :show-edge="false"
                  style="width: 100%; font-size: 18px;"
                >
                  <template #suffix>¥</template>
                </n-input-number>
              </n-space>

              <n-divider style="margin: 8px 0;" />

              <n-space vertical :size="12">
                <n-space justify="space-between">
                  <n-text style="color: #3D3D3D; font-size: 15px;">消费金额</n-text>
                  <n-text strong style="color: #3D3D3D; font-size: 20px;">¥ {{ consumption.toFixed(2) }}</n-text>
                </n-space>
                <n-space justify="space-between">
                  <n-text style="color: #3D3D3D; font-size: 15px;">剩余金额</n-text>
                  <n-text strong :style="{ color: remaining < 0 ? '#D96E6E' : '#3D3D3D', fontSize: '20px' }">¥ {{ remaining.toFixed(2) }}</n-text>
                </n-space>
              </n-space>

              <n-divider style="margin: 8px 0;" />

              <n-button
                type="warning"
                block
                size="large"
                style="background: #E8985E; color: #FFFFFF; border: none; font-size: 16px;"
                @click="handleDeduct"
              >
                💸 扣费
              </n-button>

              <n-space align="center" :size="16">
                <n-checkbox v-model:checked="clearBalance" style="font-size: 15px;">
                  清空余额
                </n-checkbox>
                <n-button type="error" size="medium" @click="handleClear">
                  🗑 清空
                </n-button>
              </n-space>
            </n-space>
          </n-card>
        </div>
      </div>
    </n-layout-content>
  </n-layout>
</template>

<script>
import { h } from 'vue'
import { NButton, NCard, NDataTable, NInputNumber, NCheckbox, NLayout, NLayoutHeader, NLayoutContent, NSpace, NText, NDivider } from 'naive-ui'
import TimePicker from './TimePicker.vue'

export default {
  components: {
    TimePicker
  }
}
</script>
