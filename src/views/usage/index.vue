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
      <el-table-column prop="peer_id" :label="T('Peer')" min-width="100" />
      <el-table-column prop="from_peer" :label="T('FromPeer')" min-width="100" />
      <el-table-column prop="from_name" :label="T('FromName')" min-width="150" />
      <el-table-column prop="from_ip" :label="T('FromIP')" min-width="120" />
      <el-table-column prop="uuid" :label="T('Uuid')" min-width="200" show-overflow-tooltip />
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
import { list as listAudit } from '@/api/audit'

const loading = ref(false)
const displayList = ref([])

// Кэши
const peersCache = ref(new Map())      // IP -> peer_id
const auditCache = ref(new Map())      // peer_id -> {from_peer, from_name, from_ip, uuid}

const formatSpeed = (speedStr) => {
  if (!speedStr) return 0
  return parseFloat(speedStr)
}

// Получение списка всех пиров (IP -> peer_id)
const fetchPeersList = async () => {
  try {
    const response = await listPeers({ page: 1, page_size: 1000 })
    if (response.code === 0 && response.data && response.data.list) {
      response.data.list.forEach(peer => {
        if (peer.last_online_ip) {
          let cleanIp = peer.last_online_ip.replace(/^::ffff:/, '')
          peersCache.value.set(cleanIp, peer.id)
        }
        if (peer.ip) {
          let cleanIp = peer.ip.replace(/^::ffff:/, '')
          peersCache.value.set(cleanIp, peer.id)
        }
      })
    }
  } catch (error) {
    console.error('Error fetching peers:', error)
  }
}

// Получение данных из журнала соединений по peer_id
const fetchAuditByPeerId = async (peerId) => {
  if (auditCache.value.has(peerId)) {
    return auditCache.value.get(peerId)
  }
  
  try {
    const response = await listAudit({ 
      page: 1, 
      page_size: 10,
      peer_id: peerId 
    })
    
    if (response.code === 0 && response.data && response.data.list && response.data.list.length > 0) {
      const record = response.data.list[0]
      const auditInfo = {
        from_peer: record.from_peer || '-',
        from_name: record.from_name || '-',
        from_ip: record.ip || '-',
        uuid: record.uuid || '-'
      }
      auditCache.value.set(peerId, auditInfo)
      return auditInfo
    }
  } catch (error) {
    console.error(`Error fetching audit for peer ${peerId}:`, error)
  }
  
  const defaultInfo = {
    from_peer: '-',
    from_name: '-',
    from_ip: '-',
    uuid: '-'
  }
  auditCache.value.set(peerId, defaultInfo)
  return defaultInfo
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
        
        let ip = parts[0] || '-'
        ip = ip.replace(/^::ffff:/, '')
        ip = ip.replace(/:\d+:$/, '')
        
        let time = parts[1] || '-'
        time = time.replace(/s$/, '') + ' сек'
        
        let total = parts[2] || '0'
        const totalValue = parseFloat(total)
        const totalUnit = total.includes('MB') ? 'MB' : (total.includes('KB') ? 'KB' : 'B')
        
        const peerId = peersCache.value.get(ip)
        
        let peer = '-'
        let fromPeer = '-'
        let fromName = '-'
        let fromIp = '-'
        let uuid = '-'
        
        if (peerId) {
          peer = peerId
          const auditInfo = await fetchAuditByPeerId(peerId)
          fromPeer = auditInfo.from_peer
          fromName = auditInfo.from_name
          fromIp = auditInfo.from_ip
          uuid = auditInfo.uuid
        } else {
          const auditResponse = await listAudit({ page: 1, page_size: 10, ip: ip })
          if (auditResponse.code === 0 && auditResponse.data && auditResponse.data.list && auditResponse.data.list.length > 0) {
            const record = auditResponse.data.list[0]
            peer = record.peer_id || '-'
            fromPeer = record.from_peer || '-'
            fromName = record.from_name || '-'
            fromIp = record.ip || '-'
            uuid = record.uuid || '-'
          }
        }
        
        connections.push({
          ip: ip,
          peer_id: peer,
          from_peer: fromPeer,
          from_name: fromName,
          from_ip: fromIp,
          uuid: uuid,
          time: time,
          total: totalValue,
          total_unit: totalUnit,
          max_speed: formatSpeed(parts[3]),
          avg_speed: formatSpeed(parts[4]),
          current_speed: formatSpeed(parts[5])
        })
      }
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
