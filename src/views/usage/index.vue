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
      <el-table-column prop="peer" :label="T('Peer')" min-width="100" />
      <el-table-column prop="from_peer" :label="T('FromPeer')" min-width="100" />
      <el-table-column prop="from_name" :label="T('FromName')" min-width="150" />
      <el-table-column prop="time" :label="T('Time')" min-width="100" />
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

const loading = ref(false)
const displayList = ref([])
const peersCache = ref(new Map())

const fetchPeersList = async () => {
  try {
    const response = await listPeers({ page: 1, page_size: 1000 })
    if (response.code === 0 && response.data && response.data.list) {
      const cache = new Map()
      response.data.list.forEach(peer => {
        if (peer.ip) {
          // Очищаем IP от префикса ::ffff:
          let cleanIp = peer.ip.replace(/^::ffff:/, '')
          cache.set(cleanIp, {
            peer_id: peer.id,
            from_peer: peer.peer_id || peer.id,
            from_name: peer.hostname || peer.alias || peer.name || peer.id
          })
        }
      })
      peersCache.value = cache
    }
  } catch (error) {
    console.error('Error fetching peers:', error)
  }
}

const fetchUsage = async () => {
  try {
    const res = await sendCmd({ cmd: 'u', target: RELAY_TARGET })
    if (res && res.data) {
      const lines = res.data.split('\n').filter(line => line.trim())
      const connections = lines.map(line => {
        const parts = line.trim().split(/\s+/)
        
        // Очищаем IP
        let ip = parts[0] || '-'
        ip = ip.replace(/^::ffff:/, '')
        ip = ip.replace(/:\d+:$/, '')
        
        // Время в секундах
        let time = parts[1] || '-'
        time = time.replace(/s$/, '') + ' сек'
        
        // Объём данных
        let total = parts[2] || '0'
        const totalValue = parseFloat(total)
        const totalUnit = total.includes('MB') ? 'MB' : (total.includes('KB') ? 'KB' : 'B')
        
        // Скорости
        const parseSpeed = (speedStr) => {
          if (!speedStr) return 0
          return parseFloat(speedStr)
        }
        
        const peerInfo = peersCache.value.get(ip) || {}
        
        return {
          ip: ip,
          peer: peerInfo.peer_id || '-',
          from_peer: peerInfo.from_peer || '-',
          from_name: peerInfo.from_name || '-',
          time: time,
          total: totalValue,
          total_unit: totalUnit,
          max_speed: parseSpeed(parts[3]),
          avg_speed: parseSpeed(parts[4]),
          current_speed: parseSpeed(parts[5])
        }
      })
      displayList.value = connections
    }
  } catch (error) {
    console.error('Error fetching usage:', error)
    ElMessage.error(T('DataLoadError'))
  }
}

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
