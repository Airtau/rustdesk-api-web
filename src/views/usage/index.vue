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
      <el-table-column prop="full_id" :label="T('ConnectionId')" min-width="180" show-overflow-tooltip />
      <el-table-column prop="ip" :label="T('IP')" min-width="120" />
      <el-table-column prop="port" :label="T('Port')" min-width="80" />
      <el-table-column prop="peer_id" :label="T('Peer')" min-width="100" />
      <el-table-column prop="from_peer" :label="T('FromPeer')" min-width="100" />
      <el-table-column prop="from_name" :label="T('FromName')" min-width="150" />
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

// Кэш для аудита по времени
const auditCache = ref(new Map())

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
  return Date.now() // fallback
}

// Поиск в журнале соединений по IP и времени
const findAuditByIpAndTime = async (ip, secondsAgo, port) => {
  const cacheKey = `${ip}:${secondsAgo}:${port}`
  
  if (auditCache.value.has(cacheKey)) {
    return auditCache.value.get(cacheKey)
  }
  
  try {
    // Получаем текущее время сервера
    const now = await getServerTime()
    const createdAfter = new Date(now - (secondsAgo + 10) * 1000) // запас 10 секунд
    const createdBefore = new Date(now - (secondsAgo - 10) * 1000)
    
    // Ищем записи за последние 5 минут
    const response = await listAudit({ 
      page: 1, 
      page_size: 50,
      ip: ip
    })
    
    if (response.code === 0 && response.data && response.data.list) {
      // Ищем запись с ближайшим created_at
      let bestMatch = null
      let minDiff = Infinity
      
      for (const record of response.data.list) {
        const recordTime = new Date(record.created_at).getTime()
        const expectedTime = now - secondsAgo * 1000
        const diff = Math.abs(recordTime - expectedTime)
        
        // Также проверяем by peer_id
        if (diff < minDiff && diff < 60000) { // разница не более 60 секунд
          minDiff = diff
          bestMatch = record
        }
      }
      
      if (bestMatch) {
        const auditInfo = {
          peer_id: bestMatch.peer_id || '-',
          from_peer: bestMatch.from_peer || '-',
          from_name: bestMatch.from_name || '-',
          uuid: bestMatch.uuid || '-',
          created_at: bestMatch.created_at || '-'
        }
        auditCache.value.set(cacheKey, auditInfo)
        return auditInfo
      }
    }
  } catch (error) {
    console.error(`Error finding audit for ${ip}:${secondsAgo}s`, error)
  }
  
  const defaultInfo = {
    peer_id: '-',
    from_peer: '-',
    from_name: '-',
    uuid: '-',
    created_at: '-'
  }
  auditCache.value.set(cacheKey, defaultInfo)
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
        
        // Полный идентификатор: ::ffff:85.114.8.78:49421
        const fullId = parts[0] || '-'
        
        // Извлекаем IP и порт
        let ipPort = fullId.replace(/^::ffff:/, '')
        let ip = ipPort
        let port = ''
        
        const lastColon = ipPort.lastIndexOf(':')
        if (lastColon !== -1) {
          ip = ipPort.substring(0, lastColon)
          port = ipPort.substring(lastColon + 1)
        }
        
        // Время в секундах
        let timeStr = parts[1] || '0'
        const secondsAgo = parseInt(timeStr.replace(/s$/, ''))
        let timeDisplay = secondsAgo + ' сек'
        
        // Объём данных
        let total = parts[2] || '0'
        const totalValue = parseFloat(total)
        const totalUnit = total.includes('MB') ? 'MB' : (total.includes('KB') ? 'KB' : 'B')
        
        // Ищем в журнале по IP и времени
        const auditInfo = await findAuditByIpAndTime(ip, secondsAgo, port)
        
        connections.push({
          full_id: fullId,
          ip: ip,
          port: port,
          peer_id: auditInfo.peer_id,
          from_peer: auditInfo.from_peer,
          from_name: auditInfo.from_name,
          uuid: auditInfo.uuid,
          created_at: auditInfo.created_at,
          time: timeDisplay,
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
    // Очищаем кэш при каждом обновлении
    auditCache.value.clear()
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
