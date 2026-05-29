<script setup>
import { computed, onBeforeUnmount, onMounted, ref, watch } from 'vue'
import CesiumMap from './data-screen/CesiumMap.vue'
import ElectronicFence from './data-screen/functions/8_ElectronicFence.vue'

const now = ref(new Date())
let clockTimer = null

const isOnline = ref(navigator.onLine)
const cesiumMapRef = ref(null)
const fencePanelRef = ref(null)

const props = defineProps({
  routeData: {
    type: Array,
    default: () => []
  },
  eventVisualization: {
    type: Object,
    default: null,
  },
})

const emit = defineEmits(['switch_page', 'visualize-event', 'start-event-simulation', 'stop-event-simulation', 'anomaly-triggered'])

const isSimulationModalVisible = ref(false)
const selectedSimulationRouteIds = ref([])
const simulationTimers = ref([])
const simulationParamsMap = ref({})

const simulationParams = ref({
  user: {
    level: 14,
    eventTime: Date.now(),
    warningRadiusMeters: 500,
    maxRouteReturnGrids: 300,
    maxRouteCheckGrids: 500,
    persist: true,
    cruisingSpeed: 15,
    groundRescueSpeed: 12,
  },
  auto: {
    eventType: 'lost_contact',
    telemetry: {
      speed: 16,
      battery: 28,
      linkLostSeconds: 90,
      deviationMeters: 120,
    },
    position: [120.1234, 30.2345, 120],
    flightPlan: {
      home: [120.13, 30.22, 80],
    },
    control: {
      apply: false,
      radiusMeters: 500,
      maxGridCount: 300,
      ttlSeconds: 1800,
    },
    landingSites: [],
    rescueResources: [],
    condition: {},
  }
})

const networkStatus = computed(() => {
  return isOnline.value ? '运行中' : '网络未连接'
})

const networkStatusClass = computed(() => {
  return isOnline.value ? 'online' : 'offline'
})

const timeText = computed(() => {
  const d = now.value
  const pad = (n) => String(n).padStart(2, '0')
  return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())} ${pad(d.getHours())}:${pad(d.getMinutes())}:${pad(d.getSeconds())}`
})

function handleOnline() {
  isOnline.value = true
}

function handleOffline() {
  isOnline.value = false
}

onMounted(() => {
  clockTimer = window.setInterval(() => {
    now.value = new Date()
  }, 1000)

  isOnline.value = navigator.onLine
  window.addEventListener('online', handleOnline)
  window.addEventListener('offline', handleOffline)
})

onBeforeUnmount(() => {
  if (clockTimer) {
    window.clearInterval(clockTimer)
    clockTimer = null
  }
  simulationTimers.value.forEach(t => window.clearTimeout(t))
  simulationTimers.value = []
  window.removeEventListener('online', handleOnline)
  window.removeEventListener('offline', handleOffline)
})

watch(() => cesiumMapRef.value?.isMapReady, (ready) => {
  if (ready) {
    console.log('[MonitoringScreen] CesiumMap 就绪')
    drawAllRoutes()
  }
})

watch(() => props.routeData, () => {
  drawAllRoutes()
}, { deep: true })

watch(() => props.eventVisualization, async (payload) => {
  if (payload) {
    await drawEventVisualization(payload)
  } else {
    clearEventVisualization()
  }
}, { deep: true, immediate: true })

function drawAllRoutes() {
  const routes = props.routeData || []
  console.log('[MonitoringScreen] 绘制航线:', routes.length)

  routes.forEach(route => {
    if (route && route.id && route.path) {
      if (cesiumMapRef.value && typeof cesiumMapRef.value.drawRouteGrid === 'function') {
        cesiumMapRef.value.drawRouteGrid(route.id, route.path)
      }
    }
  })
}

function clearRouteGrid(routeId) {
  if (cesiumMapRef.value && typeof cesiumMapRef.value.clearRouteGrid === 'function') {
    cesiumMapRef.value.clearRouteGrid(routeId)
  }
}

function clearEventVisualization() {
  if (cesiumMapRef.value && typeof cesiumMapRef.value.clearEventVisualization === 'function') {
    cesiumMapRef.value.clearEventVisualization()
  }
}

async function fetchGridBoundaryByCode(gridCode) {
  if (!gridCode) return null
  const headers = new Headers()
  headers.append('X-API-Key', import.meta.env.VITE_API_KEY || '')
  headers.append('Content-Type', 'application/json')
  const resp = await fetch('/api/multiSource/basicGrid/getGridBoundaryByCode', {
    method: 'POST',
    headers,
    body: JSON.stringify({ gridCode }),
  })
  const data = await resp.json().catch(() => null)
  if (!resp.ok || data?.status !== 'success' || !data?.data) return null
  return data.data
}

function normalizeGridBoundary(grid) {
  if (!grid) return null
  return {
    code: grid.code || grid.gridCode || '',
    center: grid.center || null,
    bounds: {
      west: grid.west,
      east: grid.east,
      south: grid.south,
      north: grid.north,
      top: grid.top ?? 0,
      bottom: grid.bottom ?? 0,
    },
  }
}

async function drawWarningSampleGrids(warningArea) {
  const codes = Array.isArray(warningArea?.sampleGrids) ? warningArea.sampleGrids : []
  const limit = Number(warningArea?.sampleGridCount) || codes.length
  const sampleCells = []

  for (const code of codes.slice(0, limit)) {
    const cell = await fetchGridBoundaryByCode(code)
    const normalized = normalizeGridBoundary(cell)
    if (normalized) sampleCells.push(normalized)
  }

  if (sampleCells.length > 0 && cesiumMapRef.value && typeof cesiumMapRef.value.drawGridBoundary === 'function') {
    cesiumMapRef.value.drawGridBoundary({
      cells: sampleCells.map(cell => ({
        code: cell.code,
        center: cell.center,
        bounds: cell.bounds,
        color: '#f59e0b',
        level: warningArea?.gridLevel,
      }))
    })
  }
}

async function drawEventVisualization(payload) {
  if (!cesiumMapRef.value) return

  if (payload?.eventGrid && typeof cesiumMapRef.value.drawGridBoundary === 'function') {
    cesiumMapRef.value.drawGridBoundary({
      code: payload.eventGrid.code,
      center: payload.eventGrid.center,
      bounds: {
        west: payload.eventGrid.minlon,
        east: payload.eventGrid.maxlon,
        south: payload.eventGrid.minlat,
        north: payload.eventGrid.maxlat,
        top: payload.eventGrid.top,
        bottom: payload.eventGrid.bottom,
      },
    })
  }

  await drawWarningSampleGrids(payload?.warningArea)

  if (typeof cesiumMapRef.value.drawEventVisualization === 'function') {
    cesiumMapRef.value.drawEventVisualization({
      eventPoint: payload?.eventPoint,
      warningArea: payload?.warningArea,
    })
  }
}

const routeStats = computed(() => {
  const routes = props.routeData || []
  return {
    total: routes.length,
    totalGrids: routes.reduce((sum, r) => sum + (r.path?.length || 0), 0)
  }
})

function handleDrawSphereFence(params) {
  if (cesiumMapRef.value && typeof cesiumMapRef.value.startDrawSphereFence === 'function') {
    cesiumMapRef.value.startDrawSphereFence(params, () => {})
  }
}

function handleDrawLineFence(params) {
  if (cesiumMapRef.value && typeof cesiumMapRef.value.startDrawLineFence === 'function') {
    cesiumMapRef.value.startDrawLineFence(params, () => {})
  }
}

function handleCompleteSphereFence(fenceData) {
  if (cesiumMapRef.value && typeof cesiumMapRef.value.completeSphereFence === 'function') {
    cesiumMapRef.value.completeSphereFence(fenceData, (data) => {
      fencePanelRef.value?.confirmFenceDraw(data)
    })
  }
}

function handleCompleteLineFence(fenceData) {
  if (cesiumMapRef.value && typeof cesiumMapRef.value.completeLineFence === 'function') {
    cesiumMapRef.value.completeLineFence(fenceData, (data) => {
      fencePanelRef.value?.confirmFenceDraw(data)
    })
  }
}

function handleClearFence() {
  if (cesiumMapRef.value && typeof cesiumMapRef.value.clearFenceDrawing === 'function') {
    cesiumMapRef.value.clearFenceDrawing()
  }
}

function handleToggleFence(fence) {
  if (cesiumMapRef.value && typeof cesiumMapRef.value.toggleFenceVisibility === 'function') {
    cesiumMapRef.value.toggleFenceVisibility(fence)
  }
}

function handleDeleteFence(id) {
  if (cesiumMapRef.value && typeof cesiumMapRef.value.removeFenceFromMap === 'function') {
    cesiumMapRef.value.removeFenceFromMap(id)
  }
}

function handleStartEventSimulation() {
  const routes = props.routeData || []
  if (routes.length === 0) return
  selectedSimulationRouteIds.value = routes.map(route => route.id)
  isSimulationModalVisible.value = true
}

function handleStopEventSimulation() {
  simulationTimers.value.forEach(t => window.clearTimeout(t))
  simulationTimers.value = []
  simulationParamsMap.value = {}

  if (cesiumMapRef.value) {
    if (typeof cesiumMapRef.value.stopUavAnimation === 'function') {
      cesiumMapRef.value.stopUavAnimation()
    }
    if (typeof cesiumMapRef.value.clearEventVisualization === 'function') {
      cesiumMapRef.value.clearEventVisualization()
    }
    if (typeof cesiumMapRef.value.clearGridVisual === 'function') {
      cesiumMapRef.value.clearGridVisual()
    }
  }

  emit('stop-event-simulation')
}

function handleCloseSimulationModal() {
  isSimulationModalVisible.value = false
}

function toggleSimulationRoute(routeId) {
  const index = selectedSimulationRouteIds.value.indexOf(routeId)
  if (index >= 0) {
    selectedSimulationRouteIds.value.splice(index, 1)
  } else {
    selectedSimulationRouteIds.value.push(routeId)
  }
}

function resetSimulationParams() {
  simulationParams.value.user = {
    level: 14,
    warningRadiusMeters: 500,
    maxRouteReturnGrids: 300,
    maxRouteCheckGrids: 500,
    persist: true,
  }
  simulationParams.value.auto = {
    eventType: 'lost_contact',
    telemetry: {
      speed: 16,
      battery: 28,
      linkLostSeconds: 90,
      deviationMeters: 120,
    },
    position: [120.1234, 30.2345, 120],
    flightPlan: {
      home: [120.13, 30.22, 80],
    },
    control: {
      apply: false,
      radiusMeters: 500,
      maxGridCount: 300,
      ttlSeconds: 1800,
    },
    landingSites: [],
    rescueResources: [],
    condition: {},
  }
}

function haversineDistanceMeters(a, b) {
  if (!a || !b) return Number.POSITIVE_INFINITY
  const toRad = deg => (deg * Math.PI) / 180
  const R = 6371000
  const dLat = toRad(b[1] - a[1])
  const dLon = toRad(b[0] - a[0])
  const lat1 = toRad(a[1])
  const lat2 = toRad(b[1])
  const sinLat = Math.sin(dLat / 2)
  const sinLon = Math.sin(dLon / 2)
  const c = 2 * Math.asin(Math.sqrt(sinLat * sinLat + Math.cos(lat1) * Math.cos(lat2) * sinLon * sinLon))
  return R * c
}

function offsetPointMeters(point, eastMeters, northMeters) {
  if (!point) return null
  const latRad = (point[1] * Math.PI) / 180
  const deltaLat = northMeters / 111320
  const deltaLon = eastMeters / (111320 * Math.cos(latRad) || 1)
  return [point[0] + deltaLon, point[1] + deltaLat, point[2] ?? 120]
}

function extractRoutePoints(route) {
  const path = Array.isArray(route?.path) ? route.path : []
  return path.flatMap(item => {
    if (Array.isArray(item) && item.length >= 2 && Number.isFinite(Number(item[0])) && Number.isFinite(Number(item[1]))) {
      return [[Number(item[0]), Number(item[1]), Number(item[2] ?? 120)]]
    }
    if (item && typeof item === 'object') {
      const lon = Number(item.lon ?? item.lng ?? item.longitude)
      const lat = Number(item.lat ?? item.latitude)
      if (Number.isFinite(lon) && Number.isFinite(lat)) {
        return [[lon, lat, Number(item.height ?? item.alt ?? item.z ?? 120)]]
      }
    }
    return []
  })
}

async function resolveRouteAnchorPoint(route) {
  const routePoints = extractRoutePoints(route)
  if (routePoints.length > 0) return routePoints[Math.floor(routePoints.length / 2)]

  const codes = Array.isArray(route?.path) ? route.path : []
  for (const code of codes) {
    const boundary = await fetchGridBoundaryByCode(code)
    const center = boundary?.center
    if (Array.isArray(center) && center.length >= 2) {
      return [Number(center[0]), Number(center[1]), Number(center[2] ?? 120)]
    }
  }
  return null
}

async function buildAutoSimulationParams(routeIds) {
  const routes = (props.routeData || []).filter(route => routeIds.includes(route.id))
  const selectedRoute = routes[0] || null
  const anchorPoint = selectedRoute ? await resolveRouteAnchorPoint(selectedRoute) : null
  const onRoute = !!anchorPoint && routeIds.length > 0
  const useOffset = onRoute && Math.random() > 0.55
  const eventPoint = useOffset ? offsetPointMeters(anchorPoint, 120, 60) : anchorPoint
  const deviationMeters = eventPoint && anchorPoint ? Math.round(haversineDistanceMeters(anchorPoint, eventPoint)) : 0
  const safeDeviation = Math.min(200, deviationMeters)
  const autoEventType = safeDeviation === 0 ? 'lost_contact' : (safeDeviation < 80 ? 'deviation' : 'low_battery')

  return {
    eventType: autoEventType,
    telemetry: {
      speed: Math.max(5, Math.min(25, 12 + Math.round(Math.random() * 6))),
      battery: Math.max(10, Math.min(100, safeDeviation === 0 ? 42 : 26)),
      linkLostSeconds: Math.max(0, Math.min(600, safeDeviation === 0 ? 30 : 90)),
      deviationMeters: safeDeviation,
    },
    position: eventPoint || [120.1234, 30.2345, 120],
    flightPlan: {
      home: [120.13, 30.22, 80],
    },
    control: {
      apply: false,
      radiusMeters: 500,
      maxGridCount: 300,
      ttlSeconds: 1800,
    },
    landingSites: [],
    rescueResources: [],
    condition: {},
  }
}

async function handleConfirmSimulation() {
  if (selectedSimulationRouteIds.value.length === 0) return

  simulationTimers.value.forEach(t => window.clearTimeout(t))
  simulationTimers.value = []
  simulationParamsMap.value = {}

  if (cesiumMapRef.value) {
    if (typeof cesiumMapRef.value.stopUavAnimation === 'function') {
      cesiumMapRef.value.stopUavAnimation()
    }
    if (typeof cesiumMapRef.value.clearEventVisualization === 'function') {
      cesiumMapRef.value.clearEventVisualization()
    }
    if (typeof cesiumMapRef.value.clearGridVisual === 'function') {
      cesiumMapRef.value.clearGridVisual()
    }
  }

  const routeIds = [...selectedSimulationRouteIds.value]
  const routes = (props.routeData || []).filter(route => routeIds.includes(route.id))

  const allAutoParams = {}
  for (const routeId of routeIds) {
    const auto = await buildAutoSimulationParams([routeId])
    allAutoParams[routeId] = auto
    simulationParamsMap.value[routeId] = {
      user: { ...simulationParams.value.user },
      auto,
    }
  }

  const firstAuto = allAutoParams[routeIds[0]]
  emit('start-event-simulation', {
    routeIds,
    params: JSON.parse(JSON.stringify({
      user: { ...simulationParams.value.user },
      auto: firstAuto,
    }))
  })

  routes.forEach((route, index) => {
    if (cesiumMapRef.value && typeof cesiumMapRef.value.startUavAnimation === 'function') {
      cesiumMapRef.value.startUavAnimation(route.path, route.id, index === 0)
    }

    const delay = 3000 + Math.random() * 5000
    const timer = window.setTimeout(() => {
      const willTrigger = Math.random() < 0.8
      if (willTrigger) {
        triggerAnomaly(route.id)
      }
    }, delay)
    simulationTimers.value.push(timer)
  })

  isSimulationModalVisible.value = false
}

async function triggerAnomaly(routeId) {
  if (!cesiumMapRef.value) return

  const currentPosition = typeof cesiumMapRef.value.getUavCurrentPosition === 'function'
    ? cesiumMapRef.value.getUavCurrentPosition(routeId)
    : null

  if (typeof cesiumMapRef.value.stopUavAnimationByRouteId === 'function') {
    cesiumMapRef.value.stopUavAnimationByRouteId(routeId)
  }

  const params = simulationParamsMap.value[routeId]
  if (!params) return

  const position = currentPosition || params.auto.position

  const requestBody = {
    position,
    eventType: params.auto.eventType,
    level: params.user.level,
    eventTime: params.user.eventTime || Date.now(),
    telemetry: params.auto.telemetry,
    cruisingSpeed: params.user.cruisingSpeed,
    groundRescueSpeed: params.user.groundRescueSpeed,
    flightPlan: params.auto.flightPlan,
    landingSites: params.auto.landingSites,
    rescueResources: params.auto.rescueResources,
    condition: params.auto.condition,
    control: params.auto.control,
    warningRadiusMeters: params.user.warningRadiusMeters,
    maxRouteReturnGrids: params.user.maxRouteReturnGrids,
    maxRouteCheckGrids: params.user.maxRouteCheckGrids,
    persist: params.user.persist,
  }

  try {
    const headers = new Headers()
    headers.append('Content-Type', 'application/json')
    headers.append('X-API-Key', import.meta.env.VITE_API_KEY || '')

    const resp = await fetch('/api/emergency/handle', {
      method: 'POST',
      headers,
      body: JSON.stringify(requestBody),
    })

    const data = await resp.json()
    if (data?.status === 'success' && data?.data) {
      await visualizeEmergencyResponse(data.data)
      emit('anomaly-triggered', {
        routeId,
        routeName: (props.routeData || []).find(r => r.id === routeId)?.name || '',
        eventType: params.auto.eventType,
        position,
        requestBody,
        responseData: data.data,
      })
    } else {
      console.warn('[MonitoringScreen] 应急接口返回异常:', data?.message)
    }
  } catch (err) {
    console.error('[MonitoringScreen] 应急接口调用失败:', err)
  }
}

async function visualizeEmergencyResponse(responseData) {
  if (!cesiumMapRef.value) return

  const allCells = []

  if (responseData.eventGrid) {
    allCells.push({
      code: responseData.eventGrid.code,
      center: responseData.eventGrid.center,
      bounds: {
        west: responseData.eventGrid.minlon,
        east: responseData.eventGrid.maxlon,
        south: responseData.eventGrid.minlat,
        north: responseData.eventGrid.maxlat,
        top: responseData.eventGrid.top,
        bottom: responseData.eventGrid.bottom,
      },
      color: '#ef4444',
      level: responseData.warningArea?.gridLevel,
    })
  }

  const warningArea = responseData.warningArea
  if (warningArea && Array.isArray(warningArea.sampleGrids)) {
    const codes = warningArea.sampleGrids
    const limit = Number(warningArea.sampleGridCount) || codes.length
    for (const code of codes.slice(0, limit)) {
      const cell = await fetchGridBoundaryByCode(code)
      const normalized = normalizeGridBoundary(cell)
      if (normalized) {
        allCells.push({
          code: normalized.code,
          center: normalized.center,
          bounds: normalized.bounds,
          color: '#f59e0b',
          level: warningArea.gridLevel,
        })
      }
    }
  }

  if (allCells.length > 0 && typeof cesiumMapRef.value.drawGridBoundary === 'function') {
    cesiumMapRef.value.drawGridBoundary({ cells: allCells })
  }

  if (typeof cesiumMapRef.value.drawEventVisualization === 'function') {
    const eg = responseData.eventGrid
    cesiumMapRef.value.drawEventVisualization({
      eventPoint: eg?.center ? {
        lon: eg.center[0],
        lat: eg.center[1],
        height: eg.center[2] || 0,
      } : null,
      warningArea: warningArea || null,
    })
  }
}

function handleSaveFence(fences) {
  console.log('保存围栏到后端:', fences)
}

function handleDrawFence(fence) {
  if (cesiumMapRef.value && typeof cesiumMapRef.value.drawFenceOnMap === 'function') {
    cesiumMapRef.value.drawFenceOnMap(fence)
  }
}

const fenceStats = computed(() => {
  const fences = fencePanelRef.value?.fences || []
  const applied = fences.filter(f => f.enabled).length
  return {
    total: fences.length,
    applied: applied,
    unapplied: fences.length - applied
  }
})

</script>

<template>
  <div class="app-root">
    <CesiumMap ref="cesiumMapRef" :show3-d-toggle="false" />

    <!-- 顶部导航 -->
    <header class="topbar">
      <!-- 左侧导航按钮 -->
      <div class="left-nav-buttons">
        <div class="nav-btn" @click="$emit('switch_page', 'grid')">网格化算子</div>
        <div class="nav-btn active">实时监控大屏</div>
        <div class="nav-btn" @click="$emit('switch_page', 'info')">信息管理系统</div>
      </div>

      <!-- 中间标题 -->
      <div class="topbar-center">
        <h1 class="main-title">低空路径智绘平台</h1>
      </div>

      <!-- 右侧状态和时间 -->
      <div class="topbar-right">
        <div class="status-box">
          <span>系统状态：</span>
          <span :class="['status-value', networkStatusClass]">{{ networkStatus }}</span>
        </div>
        <div class="time-box">
          <span>{{ timeText }}</span>
        </div>
      </div>
    </header>

    <!-- 实时监控大屏内容区域 -->
    <main class="monitoring-content">
      <!-- 左侧电子围栏面板 -->
      <div class="control-panel-fence">
        <div class="fence-panel-wrapper">
          <ElectronicFence
            ref="fencePanelRef"
            :map-ready="!!cesiumMapRef"
            @draw-sphere-fence="handleDrawSphereFence"
            @draw-line-fence="handleDrawLineFence"
            @complete-sphere-fence="handleCompleteSphereFence"
            @complete-line-fence="handleCompleteLineFence"
            @clear-fence="handleClearFence"
            @toggle-fence="handleToggleFence"
            @delete-fence="handleDeleteFence"
            @save-fence="handleSaveFence"
            @draw-fence="handleDrawFence"
          />
        </div>

        <!-- 电子围栏统计信息 -->
        <div class="fence-stats">
          <div class="stats-title">电子围栏统计</div>
          <div class="stats-grid">
            <div class="stat-item">
              <div class="stat-value">{{ fenceStats.total }}</div>
              <div class="stat-label">已存储</div>
            </div>
            <div class="stat-item">
              <div class="stat-value">{{ fenceStats.applied }}</div>
              <div class="stat-label">已应用</div>
            </div>
            <div class="stat-item">
              <div class="stat-value">{{ fenceStats.unapplied }}</div>
              <div class="stat-label">未应用</div>
            </div>
          </div>
        </div>
      </div>

      <!-- 右侧监控航线面板 -->
      <div class="control-panel-route">
        <!-- 航线列表区域 -->
        <div class="route-list-section">
          <div class="panel-header">
            <span class="panel-title">已展示航线</span>
            <div class="route-header-actions">
              <button class="action-btn simulate-btn" type="button" @click="handleStartEventSimulation">异常模拟</button>
              <button class="action-btn exit-btn" type="button" @click="handleStopEventSimulation">退出</button>
            </div>
          </div>
          <div class="route-list">
            <div v-if="routeStats.total === 0" class="empty-list">
              <span>暂无监控航线</span>
            </div>
            <div
              v-for="route in props.routeData"
              :key="route.id"
              class="route-item"
            >
              <div class="route-info">
                <span class="route-name">{{ route.name || '航线 ' + route.id }}</span>
                <span class="route-id">ID: {{ route.id }}</span>
              </div>
              <div class="route-grids">
                <span class="grid-count">{{ route.path?.length || 0 }}</span>
                <span class="grid-label">格网</span>
              </div>
            </div>
          </div>
        </div>

        <!-- 航线统计区域 -->
        <div class="route-stats-section">
          <div class="stats-title">航线统计</div>
          <div class="stats-grid">
            <div class="stat-item">
              <div class="stat-value">{{ routeStats.total }}</div>
              <div class="stat-label">监控航线</div>
            </div>
            <div class="stat-item">
              <div class="stat-value">{{ routeStats.totalGrids }}</div>
              <div class="stat-label">格网总数</div>
            </div>
          </div>
        </div>
      </div>
    </main>

    <div v-if="isSimulationModalVisible" class="simulation-modal-mask" @click.self="handleCloseSimulationModal">
      <div class="simulation-modal" role="dialog" aria-modal="true" aria-labelledby="simulation-modal-title">
        <div class="simulation-modal-header">
          <div id="simulation-modal-title" class="simulation-modal-title">异常模拟参数设置</div>
          <button class="modal-btn secondary" type="button" @click="handleCloseSimulationModal">关闭</button>
        </div>

        <div class="simulation-modal-body">
          <div class="simulation-section">
            <div class="section-title">1. 选择用于异常模拟的航线</div>
            <div v-if="routeStats.total === 0" class="param-tip">当前没有已展示航线，无法执行异常模拟。</div>
            <div v-else class="route-select-grid">
              <label
                v-for="route in props.routeData"
                :key="route.id"
                class="route-select-item"
                :class="{ selected: selectedSimulationRouteIds.includes(route.id) }"
              >
                <input
                  type="checkbox"
                  :checked="selectedSimulationRouteIds.includes(route.id)"
                  @change="toggleSimulationRoute(route.id)"
                />
                <div class="route-select-main">
                  <span class="route-select-name">{{ route.name || '航线 ' + route.id }}</span>
                  <span class="route-select-meta">ID: {{ route.id }} · 格网 {{ route.path?.length || 0 }}</span>
                </div>
              </label>
            </div>
            <div class="param-tip">支持选择一条或多条航线参与异常模拟；没有航线时模拟按钮不可执行。</div>
          </div>

          <div class="simulation-section">
            <div class="section-title">2. 用户手动输入参数</div>
            <div class="param-grid single-grid">
              <div class="param-field">
                <label>网格层级</label>
                <input v-model.number="simulationParams.user.level" type="number" min="1" max="21" />
              </div>
              <div class="param-field">
                <label>警戒半径 (m)</label>
                <input v-model.number="simulationParams.user.warningRadiusMeters" type="number" min="0" />
              </div>
              <div class="param-field">
                <label>返航格网上限</label>
                <input v-model.number="simulationParams.user.maxRouteReturnGrids" type="number" min="1" />
              </div>
              <div class="param-field">
                <label>路径检查格网上限</label>
                <input v-model.number="simulationParams.user.maxRouteCheckGrids" type="number" min="1" />
              </div>
              <div class="param-field">
                <label>是否落库</label>
                <select v-model="simulationParams.user.persist">
                  <option :value="true">true</option>
                  <option :value="false">false</option>
                </select>
              </div>
            </div>
            <div class="param-tip">事件时间、巡航速度、地面救援速度、事件类型、电量、失联秒数、偏航距离等均由系统自动模拟，不在此窗口中展示。</div>
          </div>
        </div>

        <div class="simulation-modal-footer">
          <button class="modal-btn secondary" type="button" @click="resetSimulationParams">重置默认值</button>
          <button class="modal-btn secondary" type="button" @click="handleCloseSimulationModal">取消</button>
          <button class="modal-btn primary" type="button" :disabled="routeStats.total === 0 || selectedSimulationRouteIds.length === 0" @click="handleConfirmSimulation">确认</button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.app-root {
  position: relative;
  width: 100%;
  height: 100vh;
  overflow: hidden;
}

/* 顶部导航 */
.topbar {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  height: 50px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 20px;
  background: #1a365d;
  border-bottom: 2px solid #2c5282;
  z-index: 100;
}

/* 导航按钮 */
.left-nav-buttons {
  display: flex;
  gap: 4px;
}

.nav-btn {
  padding: 6px 16px;
  background: #2c5282;
  border: 1px solid #3182ce;
  color: #fff;
  font-size: 13px;
  cursor: pointer;
}

.nav-btn:hover {
  background: #3182ce;
}

.nav-btn.active {
  background: #c05621;
  border-color: #dd6b20;
}

/* 中间标题区域 */
.topbar-center {
  position: absolute;
  left: 50%;
  transform: translateX(-50%);
}

.main-title {
  font-size: 18px;
  font-weight: bold;
  color: #fff;
  margin: 0;
  letter-spacing: 2px;
}

/* 右侧状态和时间 */
.topbar-right {
  display: flex;
  align-items: center;
  gap: 20px;
}

.status-box {
  font-size: 12px;
  color: #a0aec0;
}

.status-value {
  font-weight: bold;
}

.status-value.online {
  color: #68d391;
}

.status-value.offline {
  color: #fc8181;
}

.time-box {
  padding: 4px 12px;
  background: #2d3748;
  border: 1px solid #4a5568;
  color: #e2e8f0;
  font-size: 12px;
  font-family: monospace;
}

/* 监控内容区域 */
.monitoring-content {
  position: fixed;
  top: 50px;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 50;
  pointer-events: none;
  display: flex;
  gap: 0;
}

/* 电子围栏控制面板 */
.control-panel-fence {
  width: 320px;
  height: 100%;
  background: transparent;
  pointer-events: auto;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  gap: 0;
}

.fence-panel-wrapper {
  flex: 3;
  overflow: hidden;
  position: relative;
  border-radius: 0;
  box-shadow: none;
}

.fence-panel-wrapper::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(to right, rgba(180, 200, 220, 0.92) 0%, rgba(255, 255, 255, 0.5) 50%, rgba(255, 255, 255, 0.25) 100%);
  border-radius: 0;
  pointer-events: none;
  z-index: 0;
}

.fence-panel-wrapper > * {
  position: relative;
  z-index: 1;
}

.fence-stats {
  flex: 1;
  min-height: 80px;
  padding: 12px;
  display: flex;
  flex-direction: column;
  position: relative;
  background: linear-gradient(to right, rgba(180, 200, 220, 0.92) 0%, rgba(255, 255, 255, 0.5) 50%);
  border-radius: 0;
  margin: 0;
  box-shadow: none;
}

.fence-stats > * {
  position: relative;
  z-index: 1;
}

.stats-title {
  font-size: 13px;
  font-weight: 600;
  color: #2d3748;
  margin-bottom: 10px;
  padding-bottom: 8px;
  border-bottom: 1px solid rgba(226, 232, 240, 0.6);
}

.stats-grid {
  display: flex;
  flex: 1;
  gap: 8px;
}

.stat-item {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: rgba(247, 250, 252, 0.3);
  border-radius: 8px;
  padding: 8px 4px;
  border: 1px solid rgba(226, 232, 240, 0.4);
}

.stat-value {
  font-size: 20px;
  font-weight: bold;
  color: #2d3748;
  font-family: monospace;
}

.stat-label {
  font-size: 11px;
  color: #718096;
  margin-top: 4px;
}

/* 右侧监控航线面板 */
.control-panel-route {
  position: absolute;
  right: 0;
  top: 0;
  width: 320px;
  height: 100%;
  background: transparent;
  pointer-events: auto;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  gap: 0;
}

.route-list-section {
  flex: 3;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  position: relative;
  border-radius: 0;
  box-shadow: none;
}

.route-list-section::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(to left, rgba(180, 200, 220, 0.92) 0%, rgba(255, 255, 255, 0.5) 50%, rgba(255, 255, 255, 0.25) 100%);
  border-radius: 0;
  pointer-events: none;
  z-index: 0;
}

.route-list-section > * {
  position: relative;
  z-index: 1;
}

.panel-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px;
  border-bottom: 1px solid rgba(226, 232, 240, 0.6);
  background: transparent;
  border-radius: 0;
}

.panel-title {
  font-size: 14px;
  font-weight: 600;
  color: #2d3748;
  flex-shrink: 0;
}

.route-header-actions {
  display: flex;
  align-items: center;
  gap: 8px;
  width: 100%;
  justify-content: flex-end;
}

.action-btn {
  padding: 5px 10px;
  border: 1px solid rgba(90, 150, 200, 0.7);
  border-radius: 4px;
  background: rgba(255, 255, 255, 0.9);
  color: #1e4a6e;
  font-size: 12px;
  cursor: pointer;
  transition: all 0.2s ease;
  white-space: nowrap;
  flex-shrink: 0;
}

.action-btn:hover {
  background: #ffffff;
  border-color: #4a90c2;
  box-shadow: 0 2px 6px rgba(60, 120, 180, 0.15);
}

.simulate-btn {
  color: #b45309;
}

.exit-btn {
  color: #b91c1c;
}

.panel-count {
  font-size: 16px;
  font-weight: bold;
  color: #2b6cb0;
  font-family: monospace;
}

.simulation-modal-mask {
  position: fixed;
  inset: 0;
  background: rgba(15, 23, 42, 0.45);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  pointer-events: auto;
}

.simulation-modal {
  width: min(920px, calc(100vw - 40px));
  max-height: min(86vh, 920px);
  background: #f8fbff;
  border-radius: 14px;
  box-shadow: 0 20px 60px rgba(15, 23, 42, 0.28);
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.simulation-modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 20px;
  border-bottom: 1px solid rgba(148, 163, 184, 0.25);
  background: linear-gradient(90deg, #1a365d, #2c5282);
  color: #fff;
}

.simulation-modal-title {
  font-size: 16px;
  font-weight: 700;
}

.simulation-modal-body {
  padding: 18px 20px 10px;
  overflow: auto;
}

.simulation-section {
  margin-bottom: 18px;
  padding: 14px;
  border: 1px solid rgba(148, 163, 184, 0.2);
  border-radius: 12px;
  background: #fff;
}

.section-title {
  font-size: 14px;
  font-weight: 700;
  color: #1e3a5f;
  margin-bottom: 10px;
}

.route-select-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 10px;
}

.route-select-item {
  border: 1px solid rgba(148, 163, 184, 0.24);
  border-radius: 10px;
  padding: 10px 12px;
  background: #f8fafc;
  display: flex;
  gap: 8px;
  cursor: pointer;
  transition: all 0.2s ease;
}

.route-select-item.selected {
  border-color: #3b82f6;
  background: #eff6ff;
}

.route-select-item.disabled {
  opacity: 0.65;
  cursor: not-allowed;
}

.route-select-main {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.route-select-name {
  font-size: 13px;
  font-weight: 600;
  color: #1e293b;
}

.route-select-meta {
  font-size: 12px;
  color: #64748b;
}

.param-columns {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 14px;
}

.param-group {
  border: 1px solid rgba(148, 163, 184, 0.18);
  border-radius: 10px;
  padding: 12px;
  background: #f8fafc;
}

.param-group-title {
  font-size: 13px;
  font-weight: 700;
  color: #334155;
  margin-bottom: 10px;
}

.param-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 10px;
}

.param-grid.single-grid {
  grid-template-columns: repeat(2, minmax(0, 1fr));
}

.param-grid.single-grid .param-field:last-child {
  grid-column: span 2;
}

.param-field {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.param-field label {
  font-size: 12px;
  color: #475569;
}

.param-field input,
.param-field select,
.param-field textarea {
  width: 100%;
  border: 1px solid #cbd5e1;
  border-radius: 8px;
  padding: 8px 10px;
  font-size: 13px;
  background: #fff;
  color: #0f172a;
  outline: none;
}

.param-field textarea {
  min-height: 84px;
  resize: vertical;
}

.param-tip {
  margin-top: 8px;
  font-size: 12px;
  color: #64748b;
  line-height: 1.6;
}

.simulation-modal-footer {
  padding: 14px 20px 18px;
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  border-top: 1px solid rgba(148, 163, 184, 0.22);
  background: #fff;
}

.modal-btn {
  min-width: 88px;
  padding: 9px 16px;
  border-radius: 8px;
  border: 1px solid transparent;
  cursor: pointer;
  font-size: 13px;
}

.modal-btn.secondary {
  background: #fff;
  border-color: #cbd5e1;
  color: #334155;
}

.modal-btn.primary {
  background: #2563eb;
  color: #fff;
}

.route-list {
  flex: 1;
  overflow-y: auto;
  padding: 8px;
  background: transparent;
}

.route-list::-webkit-scrollbar {
  width: 6px;
}

.route-list::-webkit-scrollbar-track {
  background: rgba(237, 242, 247, 0.5);
}

.route-list::-webkit-scrollbar-thumb {
  background: rgba(203, 213, 224, 0.6);
  border-radius: 3px;
}

.empty-list {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  color: #a0aec0;
  font-size: 12px;
}

.route-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px;
  margin-bottom: 6px;
  background: rgba(247, 250, 252, 0.3);
  border-radius: 8px;
  border: 1px solid rgba(226, 232, 240, 0.4);
  transition: all 0.2s ease;
}

.route-item:hover {
  background: rgba(247, 250, 252, 0.5);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
}

.route-info {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.route-name {
  font-size: 13px;
  color: #2d3748;
  font-weight: 500;
}

.route-id {
  font-size: 11px;
  color: #718096;
}

.route-grids {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 2px;
}

.grid-count {
  font-size: 16px;
  font-weight: bold;
  color: #2b6cb0;
  font-family: monospace;
}

.grid-label {
  font-size: 10px;
  color: #718096;
}

.route-stats-section {
  flex: 1;
  min-height: 80px;
  padding: 12px;
  display: flex;
  flex-direction: column;
  position: relative;
  background: linear-gradient(to left, rgba(180, 200, 220, 0.92) 0%, rgba(255, 255, 255, 0.5) 50%);
  border-radius: 0;
  margin: 0;
  box-shadow: none;
}

.route-stats-section > * {
  position: relative;
  z-index: 1;
}

.route-stats-section .stats-title {
  font-size: 13px;
  font-weight: 600;
  color: #2d3748;
  margin-bottom: 10px;
  padding-bottom: 8px;
  border-bottom: 1px solid rgba(226, 232, 240, 0.6);
  background: transparent;
}

.route-stats-section .stats-grid {
  flex: 1;
  display: flex;
  gap: 10px;
}

.route-stats-section .stat-item {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: rgba(247, 250, 252, 0.3);
  border-radius: 8px;
  padding: 8px 4px;
  border: 1px solid rgba(226, 232, 240, 0.4);
}

.route-stats-section .stat-value {
  font-size: 18px;
  font-weight: bold;
  color: #2b6cb0;
  font-family: monospace;
}

.route-stats-section .stat-label {
  font-size: 11px;
  color: #718096;
  margin-top: 4px;
}
</style>
