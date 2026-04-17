<template>
  <el-card class="usage-card" shadow="hover" v-loading="loading">
    <template #header>
      <div class="card-header">
        <span>{{ T('ActiveConnections') }}</span>
        <el-button type="primary" size="small" @click="getList">
          {{ T('Refresh') }}
        </el-button>
      </div>
    </template>
    <el-table :data="displayList" size="small" border stripe v-loading="loading">
      <el-table-column prop="ip" :label="T('IP')" min-width="120" />
      <el-table-column prop="port" :label="T('Port')" min-width="80" />
      <el-table-column prop="peer_id" :label="T('Peer')" min-width="100" />
      <el-table-column prop="hostname" :label="T('Hostname')" min-width="150" />
      <el-table-column prop="from_peer" :label="T('FromPeer')" min-width="100" />
      <el-table-column prop="from_name" :label="T('FromName')" min-width="150" />
      <el-table-column prop="from_ip" :label="T('FromIP')" min-width="120" />
      <el-table-column prop="uuid" :label="T('Uuid')" min-width="200" show-overflow-tooltip />
      <el-table-column prop="time" :label="T('Time')" min-width="100" />
      <el-table-column prop="created_at" :label="T('CreatedAt')" min-width="160" />
      <el-table-column prop="total" :label="T('Total')" min-width="100">
        <template #default="{ row }">
          {{ row.total }} {{ row.total_unit }}
        </template>
      </el-table-column>
      <el-table-column prop="max_speed" :label="T('MaxSpeed')" min-width="100">
        <template #default="{ row }">
          {{ row.max_speed }} kb/s
        </template>
      </el-table-column>
      <el-table-column prop="avg_speed" :label="T('AvgSpeed')" min-width="100">
        <template #default="{ row }">
          {{ row.avg_speed }} kb/s
        </template>
      </el-table-column>
      <el-table-column prop="current_speed" :label="T('CurrentSpeed')" min-width="120">
        <template #default="{ row }">
          {{ row.current_speed }} kb/s
        </template>
      </el-table-column>
    </el-table>
  </el-card>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { ElMessage } from 'element-plus'
import { T } from '@/utils/i18n'
import { sendCmd } from '@/api/rustdesk'
import { RELAY_TARGET } from '@/views/rustdesk/options'
import { list as listPeers } from '@/api/peer'
import { list as listAudit } from '@/api/audit'

const loading = ref(false)
const displayList = ref([])

// Кэши
const peersByExternalIp = ref(new Map())  // внешний IP -> массив {peer_id, hostname}
const auditCache = ref(new Map())         // peer_id -> массив активных соединений

const formatSpeed = (speedStr) => {
  if (!speedStr) return 0
  return parseFloat(speedStr)
}

// Получение текущего времени сервера
const getServerTime = async () => {
  try {
    const response = await fetch('/api/version')
    const date = response.headers.get('Date')
    if (date) {
      return new Date(date).getTime()
    }
  } catch (error) {
    console.error('Error getting server time:', error)
  }
  return Date.now()
}

// Получение списка всех пиров и группировка по внешнему IP
const fetchPeersList = async () => {
  try {
    const response = await listPeers({ page: 1, page_size: 1000 })
    if (response.code === 0 && response.data && response.data.list) {
      const map = new Map()
      response.data.list.forEach(peer => {
        if (peer.last_online_ip && peer.last_online_ip !== '0.0.0.0') {
          const externalIp = peer.last_online_ip.replace(/^::ffff:/, '')
          const peersList = map.get(externalIp) || []
          peersList.push({
            peer_id: peer.id,
            hostname: peer.hostname || peer.alias || peer.id
          })
          map.set(externalIp, peersList)
        }
      })
      peersByExternalIp.value = map
      console.log('Peers by external IP:', map.size)
    }
  } catch (error) {
    console.error('Error fetching peers:', error)
  }
}

// Получение активных соединений для конкретного peer_id из аудита
const fetchActiveAuditByPeerId = async (peerId) => {
  if (auditCache.value.has(peerId)) {
    return auditCache.value.get(peerId)
  }
  
  try {
    // Ищем активные соединения (close_time = 0)
    const response = await listAudit({ 
      page: 1, 
      page_size: 50,
      peer_id: peerId
    })
    
    const activeConnections = []
    if (response.code === 0 && response.data && response.data.list) {
      for (const record of response.data.list) {
        // close_time = 0 означает активное соединение
        if (record.close_time === 0 || record.close_time === '0') {
          activeConnections.push({
            peer_id: record.peer_id,
            from_peer: record.from_peer || '-',
            from_name: record.from_name || '-',
            from_ip: record.ip || '-',
            uuid: record.uuid || '-',
            created_at: record.created_at || '-'
          })
        }
      }
    }
    auditCache.value.set(peerId, activeConnections)
    return activeConnections
  } catch (error) {
    console.error(`Error fetching active audit for peer ${peerId}:`, error)
  }
  return []
}

// Сопоставление соединения из usage с активным аудитом
const matchWithAudit = (ip, secondsAgo, possiblePeerIds, currentTime) => {
  const expectedTime = currentTime - secondsAgo * 1000
  const TIME_TOLERANCE = 30000 // 30 секунд погрешности
  
  let bestMatch = null
  let minDiff = Infinity
  
  for (const peerId of possiblePeerIds) {
    const activeConnections = auditCache.value.get(peerId) || []
    
    for (const conn of activeConnections) {
      if (conn.created_at && conn.created_at !== '-') {
        const connTime = new Date(conn.created_at).getTime()
        const diff = Math.abs(connTime - expectedTime)
        
        if (diff < minDiff && diff < TIME_TOLERANCE) {
          minDiff = diff
          bestMatch = {
            ...conn,
            peer_id: peerId,
            hostname: possiblePeerIds.find(p => p.peer_id === peerId)?.hostname || peerId
          }
        }
      }
    }
  }
  
  return bestMatch
}

// Получение активных соединений через команду usage
const fetchUsage = async () => {
  try {
    const res = await sendCmd({ cmd: 'u', target: RELAY_TARGET })
    if (res && res.data) {
      const lines = res.data.split('\n').filter(line => line.trim())
      const connections = []
      
      for (const line of lines) {
        const parts = line.trim().split(/\s+/)
        
        // Полный идентификатор: ::ffff:85.114.8.78:49421
        const fullId = parts[0] || '-'
        
        // Извлекаем чистый IP (без порта и без ::ffff:)
        let cleanIp = fullId.replace(/^::ffff:/, '')
        const lastColon = cleanIp.lastIndexOf(':')
        let ip = cleanIp
        let port = ''
        
        if (lastColon !== -1) {
          ip = cleanIp.substring(0, lastColon)  // Чистый IP
          port = cleanIp.substring(lastColon + 1)  // Порт
        }
        
        // Время в секундах
        let timeStr = parts[1] || '0'
        const secondsAgo = parseInt(timeStr.replace(/s$/, ''))
        let timeDisplay = secondsAgo + ' сек'
        
        // Объём данных
        let total = parts[2] || '0'
        const totalValue = parseFloat(total)
        const totalUnit = total.includes('MB') ? 'MB' : (total.includes('KB') ? 'KB' : 'B')
        
        // Берём данные пира по чистому IP
        const peerInfo = peersByExternalIp.value.get(ip) || {}
        
        connections.push({
          full_id: fullId,
          ip: ip,                    // Чистый IP для отображения
          port: port,                // Порт
          peer_id: peerInfo.peer_id || '-',
          hostname: peerInfo.hostname || '-',
          username: peerInfo.username || '-',
          time: timeDisplay,
          total: totalValue,
          total_unit: totalUnit,
          max_speed: formatSpeed(parts[3]),
          avg_speed: formatSpeed(parts[4]),
          current_speed: formatSpeed(parts[5])
        })
      }
      displayList.value = connections
      console.log('Connections:', connections)
    }
  } catch (error) {
    console.error('Error fetching usage:', error)
    ElMessage.error(T('DataLoadError'))
  }
}

const getList = async () => {
  loading.value = true
  try {
    // Очищаем кэш при каждом обновлении
    auditCache.value.clear()
    await fetchPeersList()
    await fetchUsage()
    ElMessage.success(T('DataUpdated'))
  } catch (error) {
    console.error('Error:', error)
    ElMessage.error(T('DataLoadError'))
  } finally {
    loading.value = false
  }
}

onMounted(() => {
  getList()
})
</script>

<style scoped lang="scss">
.usage-card {
  width: 100%;
  margin: 0;
  
  :deep(.el-table) {
    width: 100%;
  }
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
</style>
