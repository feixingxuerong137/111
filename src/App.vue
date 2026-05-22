<script setup>
import { computed, onBeforeUnmount, ref } from 'vue'
import {
  Activity,
  AlertTriangle,
  CheckCircle2,
  Cpu,
  Gauge,
  RotateCcw,
  ShieldCheck,
  Sparkles,
  Zap,
} from 'lucide-vue-next'

const baseGpu = 'NVIDIA GeForce RTX 4070 SUPER'
const upgradedGpu = 'NVIDIA GeForce RTX 5090'

const status = ref('idle')
const progress = ref(0)
const temperature = ref(30)
const gpuName = ref(baseGpu)
const logLines = ref(['系统待命：检测到可升级显卡。'])
const activeStep = ref('等待用户授权升级显卡。')
let timerId = null

const upgradeSteps = [
  '正在扫描 PCIe 插槽',
  '正在校验显存频率',
  '正在调用量子纠缠 API',
  '正在与 NVIDIA 驱动握手',
  '正在请求 CUDA 核心借调'
]

const rollbackSteps = [
  '正在释放多余 CUDA 核心',
  '正在回收量子签名',
]

const isRunning = computed(() => status.value === 'upgrading' || status.value === 'rollingBack')
const isUpgraded = computed(() => status.value === 'upgraded')
const isRolledBack = computed(() => status.value === 'idle' || status.value === 'rolledBack')
const actionLabel = computed(() => (isUpgraded.value ? '一键回退' : '一键升级'))
const actionIcon = computed(() => (isUpgraded.value ? RotateCcw : Zap))
const actionDisabled = computed(() => isRunning.value)

const statusText = computed(() => {
  if (status.value === 'upgrading') return '升级中'
  if (status.value === 'upgraded') return '升级完成'
  if (status.value === 'rollingBack') return '回退中'
  if (status.value === 'rolledBack') return '回退成功'
  return '准备就绪'
})

const completionMessage = computed(() => {
  if (status.value === 'upgraded') return `升级完成，你当前的显卡为 ${upgradedGpu}.`
  if (status.value === 'rolledBack') return `回退成功，你当前的显卡为 ${baseGpu}.`
  return ''
})

const temperatureLevel = computed(() => {
  if (temperature.value >= 210) return 'critical'
  if (temperature.value >= 120) return 'warning'
  return 'normal'
})

const metrics = computed(() => [
  { label: '显存', value: isUpgraded.value ? '32GB GDDR7' : '12GB GDDR6X' },
  { label: '量子签名', value: isUpgraded.value ? '已通过' : '未启用' },
  { label: '风扇转速', value: temperature.value > 190 ? '∞ RPM' : `${Math.round(900 + progress.value * 24)} RPM` },
  { label: '设备管理器', value: gpuName.value },
])

function pushLog(line) {
  logLines.value = [line, ...logLines.value].slice(0, 7)
}

function runSequence(mode) {
  clearInterval(timerId)
  const upgrading = mode === 'upgrade'
  const steps = upgrading ? upgradeSteps : rollbackSteps
  status.value = upgrading ? 'upgrading' : 'rollingBack'
  progress.value = 0
  activeStep.value = steps[0]
  pushLog(upgrading ? '用户已授权：开始执行硬件升级。' : '用户已授权：开始撤销升级。')

  timerId = setInterval(() => {
    progress.value = Math.min(100, progress.value + Math.floor(Math.random() * 5) + 2)
    const stepIndex = Math.min(steps.length - 1, Math.floor((progress.value / 100) * steps.length))
    activeStep.value = steps[stepIndex]

    if (progress.value % 11 < 5) {
      pushLog(`${steps[stepIndex]}：${progress.value}%`)
    }

    if (upgrading) {
      temperature.value = Math.min(250, Math.round(30 + progress.value * 2.2))
    }

    if (progress.value >= 100) {
      clearInterval(timerId)
      timerId = null
      progress.value = 100
      if (upgrading) {
        temperature.value = 250
      }
      gpuName.value = upgrading ? upgradedGpu : baseGpu
      status.value = upgrading ? 'upgraded' : 'rolledBack'
      activeStep.value = upgrading ? '升级完成。' : '回退完成。'
      pushLog(upgrading ? `完成：当前显为 ${upgradedGpu}.` : `完成：当前显卡已恢复为 ${baseGpu}.`)
    }
  }, 180)
}

function handleAction() {
  if (isRunning.value) return
  runSequence(isUpgraded.value ? 'rollback' : 'upgrade')
}

onBeforeUnmount(() => {
  clearInterval(timerId)
})
</script>

<template>
  <main class="app-shell">
    <section class="control-panel" aria-label="GPU Quantum Upgrade">
      <div class="topbar">
        <div class="brand">
          <div class="brand-mark">
            <Cpu :size="28" />
          </div>
          <div>
            <p class="eyebrow">GPU Quantum Upgrade Console</p>
            <h1>显卡一键升级</h1>
          </div>
        </div>
        <div class="status-pill" :class="{ active: isRunning, done: isUpgraded }">
          <Activity :size="16" />
          <span>{{ statusText }}</span>
        </div>
      </div>

      <div class="main-grid">
        <section class="device-section">
          <div class="section-header">
            <ShieldCheck :size="18" />
            <span>当前设备</span>
          </div>
          <div class="gpu-readout">
            <p>Graphics Device</p>
            <strong>{{ gpuName }}</strong>
          </div>

          <div class="chip-board" aria-hidden="true">
            <div class="chip-core">
              <Sparkles :size="42" />
              <span>RTX</span>
            </div>
            <div v-for="pin in 28" :key="pin" class="pin"></div>
          </div>

          <button class="primary-action" :disabled="actionDisabled" @click="handleAction">
            <component :is="actionIcon" :size="22" />
            <span>{{ actionLabel }}</span>
          </button>

          <p v-if="completionMessage" class="completion-message">
            <CheckCircle2 :size="18" />
            <span>{{ completionMessage }}</span>
          </p>
        </section>

        <section class="telemetry-section">
          <div class="section-header">
            <Gauge :size="18" />
            <span>升级遥测</span>
          </div>

          <div class="progress-wrap">
            <div class="progress-label">
              <span>{{ activeStep }}</span>
              <strong>{{ progress }}%</strong>
            </div>
            <div class="progress-track">
              <div class="progress-fill" :style="{ width: `${progress}%` }"></div>
            </div>
          </div>

          <div class="temperature-card" :class="temperatureLevel">
            <div>
              <p>当前显卡温度</p>
              <strong>{{ temperature }}°C</strong>
            </div>
            <AlertTriangle v-if="temperatureLevel !== 'normal'" :size="30" />
          </div>

          <div class="metrics-grid">
            <div v-for="metric in metrics" :key="metric.label" class="metric-item">
              <span>{{ metric.label }}</span>
              <strong>{{ metric.value }}</strong>
            </div>
          </div>
        </section>
      </div>

      <section class="log-section" aria-label="System Log">
        <div class="section-header">
          <Activity :size="18" />
          <span>系统日志</span>
        </div>
        <div class="log-list">
          <p v-for="line in logLines" :key="line">{{ line }}</p>
        </div>
      </section>
    </section>
  </main>
</template>
