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
      <el-table-column prop="peer" :label="T('Peer')" min-width="120" />
      <el-table-column prop="from_peer" :label="T('FromPeer')" min-width="120" />
      <el-table-column prop="from_name" :label="T('FromName')" min-width="150" />
      <el-table-column prop="time" :label="T('Time')" min-width="160" />
      <el-table-column prop="total" :label="T('Total')" min-width="100">
        <template #default="{ row }">
          {{ formatBytes(row.total) }}
        </template>
      </el-table-column>
      <el-table-column prop="max_speed" :label="T('MaxSpeed')" min-width="100">
        <template #default="{ row }">
          {{ formatSpeed(row.max_speed) }}
        </template>
      </el-table-column>
      <el-table-column prop="avg_speed" :label="T('AvgSpeed')" min-width="100">
        <template #default="{ row }">
          {{ formatSpeed(row.avg_speed) }}
        </template>
      </el-table-column>
      <el-table-column prop="current_speed" :label="T('CurrentSpeed')" min-width="120">
        <template #default="{ row }">
          {{ formatSpeed(row.current_speed) }}
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

const loading = ref(false)
const displayList = ref([])

// Кэш для сопоставления IP с данными пиров
const peersCache = ref(new Map())

// Форматирование байтов в MB
const formatBytes = (bytes) => {
  if (!bytes) return '0 B'
  const mb = bytes / (1024 * 1024)
  return mb.toFixed(2) + ' MB'
}

// Форматирование скорости (байт/с -> Мбит/с)
const formatSpeed = (bytesPerSec) => {
  if (!bytesPerSec) return '0 Mbps'
  const mbps = (bytesPerSec * 8) / (1024 * 1024)
  return mbps.toFixed(2) + ' Mbps'
}

// Получение списка всех пиров из API
const fetchPeersList = async () => {
  try {
    const response = await listPeers({ page: 1, page_size: 1000 })
    
    if (response.code === 0 && response.data && response.data.list) {
      const cache = new Map()
      response.data.list.forEach(peer => {
        // Сохраняем по IP адресу
        if (peer.ip) {
          cache.set(peer.ip, {
            peer_id: peer.id,
            from_peer: peer.peer_id || peer.id,
            from_name: peer.hostname || peer.alias || peer.name || peer.id
          })
        }
        // Также сохраняем по hostname
        if (peer.hostname) {
          cache.set(peer.hostname, {
            peer_id: peer.id,
            from_peer: peer.peer_id || peer.id,
            from_name: peer.hostname
          })
        }
      })
      peersCache.value = cache
      console.log('Peers cache loaded:', cache.size)
    }
  } catch (error) {
    console.error('Error fetching peers:', error)
  }
}

// Получение активных соединений через команду usage
const fetchUsage = async () => {
  try {
    const res = await sendCmd({ cmd: 'u', target: RELAY_TARGET })
    if (res && res.data) {
      const lines = res.data.split('\n').filter(line => line.trim())
      const connections = lines.map(line => {
        const parts = line.split(/\s+/)
        const ip = parts[0]
        const peerInfo = peersCache.value.get(ip) || {}
        
        return {
          ip: ip,
          peer: peerInfo.peer_id || '-',
          from_peer: peerInfo.from_peer || '-',
          from_name: peerInfo.from_name || '-',
          time: parts[1] || '-',
          total: parseFloat(parts[2]) || 0,
          max_speed: parseFloat(parts[3]) || 0,
          avg_speed: parseFloat(parts[4]) || 0,
          current_speed: parseFloat(parts[5]) || 0
        }
      })
      displayList.value = connections
    }
  } catch (error) {
    console.error('Error fetching usage:', error)
    ElMessage.error(T('DataLoadError'))
  }
}

// Основная функция обновления
const getList = async () => {
  loading.value = true
  try {
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
