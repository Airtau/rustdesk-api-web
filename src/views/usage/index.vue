<template>
  <div class="usage-container">
    <el-card class="usage-card" shadow="hover" v-loading="loading">
      <template #header>
        <div class="card-header">
          <span>{{ T('ActiveConnections') }}</span>
          <div class="header-actions">
            <el-button :icon="Setting" @click="showColumnSetting"></el-button>
            <el-button type="primary" @click="getList">
              {{ T('Refresh') }}
            </el-button>
          </div>
        </div>
      </template>
      
      <el-table :data="displayList" size="small" border stripe v-loading="loading">
        <template v-for="col in visibleColumns.filter(c => c.visible)" :key="col.name">
          <el-table-column 
            v-if="col.name === 'ip'" 
            prop="ip" 
            :label="T('IP')" 
            :min-width="col.width" 
          />
          <el-table-column 
            v-else-if="col.name === 'port'" 
            prop="port" 
            :label="T('Port')" 
            :min-width="col.width" 
          />
          <el-table-column 
            v-else-if="col.name === 'peer_id'" 
            prop="peer_id" 
            :label="T('Peer')" 
            :min-width="col.width" 
          />
          <el-table-column 
            v-else-if="col.name === 'hostname'" 
            prop="hostname" 
            :label="T('Hostname')" 
            :min-width="col.width" 
          />
          <el-table-column 
            v-else-if="col.name === 'from_peer'" 
            prop="from_peer" 
            :label="T('FromPeer')" 
            :min-width="col.width" 
          />
          <el-table-column 
            v-else-if="col.name === 'from_name'" 
            prop="from_name" 
            :label="T('FromName')" 
            :min-width="col.width" 
          />
          <el-table-column 
            v-else-if="col.name === 'from_ip'" 
            prop="from_ip" 
            :label="T('FromIP')" 
            :min-width="col.width" 
          />
          <el-table-column 
            v-else-if="col.name === 'uuid'" 
            prop="uuid" 
            :label="T('Uuid')" 
            :min-width="col.width" 
            show-overflow-tooltip 
          />
          <el-table-column 
            v-else-if="col.name === 'time'" 
            prop="time" 
            :label="T('Time')" 
            :min-width="col.width" 
          />
          <el-table-column 
            v-else-if="col.name === 'created_at'" 
            prop="created_at" 
            :label="T('CreatedAt')" 
            :min-width="col.width" 
          />
          <el-table-column 
            v-else-if="col.name === 'total'" 
            prop="total" 
            :label="T('Total')" 
            :min-width="col.width"
          >
            <template #default="{ row }">
              {{ row.total }} {{ row.total_unit }}
            </template>
          </el-table-column>
          <el-table-column 
            v-else-if="col.name === 'max_speed'" 
            prop="max_speed" 
            :label="T('MaxSpeed')" 
            :min-width="col.width"
          >
            <template #default="{ row }">
              {{ row.max_speed }} kb/s
            </template>
          </el-table-column>
          <el-table-column 
            v-else-if="col.name === 'avg_speed'" 
            prop="avg_speed" 
            :label="T('AvgSpeed')" 
            :min-width="col.width"
          >
            <template #default="{ row }">
              {{ row.avg_speed }} kb/s
            </template>
          </el-table-column>
          <el-table-column 
            v-else-if="col.name === 'current_speed'" 
            prop="current_speed" 
            :label="T('CurrentSpeed')" 
            :min-width="col.width"
          >
            <template #default="{ row }">
              {{ row.current_speed }} kb/s
            </template>
          </el-table-column>
        </template>
      </el-table>
    </el-card>

    <!-- Диалог настройки колонок -->
    <el-dialog v-model="columnSettingVisible" :title="T('ColumnSetting')" width="500">
      <div v-for="(col, index) in allColumns" :key="col.name" style="margin-bottom: 10px; display: flex; align-items: center">
        <div style="width: 150px">
          <el-checkbox v-model="col.visible">{{ T(col.label) }}</el-checkbox>
        </div>
        <div @click="upColumn(index)" style="width: 50px; cursor: pointer">
          <el-icon><ArrowUp /></el-icon>
        </div>
        <div @click="downColumn(index)" style="width: 50px; cursor: pointer">
          <el-icon><ArrowDown /></el-icon>
        </div>
      </div>
      <template #footer>
        <el-button @click="columnSettingVisible = false">{{ T('Cancel') }}</el-button>
        <el-button type="primary" @click="saveColumnSetting">{{ T('Save') }}</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { ElMessage } from 'element-plus'
import { Setting, ArrowUp, ArrowDown } from '@element-plus/icons-vue'
import { T } from '@/utils/i18n'
import { sendCmd } from '@/api/rustdesk'
import { RELAY_TARGET } from '@/views/rustdesk/options'
import { list as listPeers } from '@/api/peer'
import { list as listAudit } from '@/api/audit'

const loading = ref(false)
const displayList = ref([])
const columnSettingVisible = ref(false)

// Все доступные колонки
const allColumns = ref([
  { name: 'ip', visible: true, label: 'IP', width: 120 },
  { name: 'port', visible: true, label: 'Port', width: 80 },
  { name: 'peer_id', visible: true, label: 'Peer', width: 100 },
  { name: 'hostname', visible: true, label: 'Hostname', width: 150 },
  { name: 'from_peer', visible: true, label: 'FromPeer', width: 100 },
  { name: 'from_name', visible: true, label: 'FromName', width: 150 },
  { name: 'from_ip', visible: true, label: 'FromIP', width: 120 },
  { name: 'uuid', visible: true, label: 'Uuid', width: 200 },
  { name: 'time', visible: true, label: 'Time', width: 100 },
  { name: 'created_at', visible: true, label: 'CreatedAt', width: 160 },
  { name: 'total', visible: true, label: 'Total', width: 100 },
  { name: 'max_speed', visible: true, label: 'MaxSpeed', width: 100 },
  { name: 'avg_speed', visible: true, label: 'AvgSpeed', width: 100 },
  { name: 'current_speed', visible: true, label: 'CurrentSpeed', width: 120 }
])

// Видимые колонки
const visibleColumns = ref([])

// Кэши
const peersByExternalIp = ref(new Map())
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
  return Date.now()
}

// Получение списка всех пиров и группировка по внешнему IP
const fetchActiveAuditByPeerId = async (peerId) => {
  if (auditCache.value.has(peerId)) {
    console.log(`  📦 [DEBUG] Кэш для peer_id=${peerId}: ${auditCache.value.get(peerId).length} соединений`)
    return auditCache.value.get(peerId)
  }
  
  console.log(`  🔍 [DEBUG] Поиск активных соединений для peer_id=${peerId}...`)
  
  try {
    // Сначала ищем по from_peer
    let response = await listAudit({ 
      page: 1, 
      page_size: 50,
      from_peer: peerId
    })
    
    let activeConnections = []
    
    if (response.code === 0 && response.data && response.data.list) {
      console.log(`  📋 [DEBUG] Найдено ${response.data.list.length} записей по from_peer`)
      
      for (const record of response.data.list) {
        const isActive = (record.action === 'new' && (record.close_time === 0 || record.close_time === '0'))
        
        console.log(`    📊 record: action=${record.action}, close_time=${record.close_time}, peer_id=${record.peer_id}, from_peer=${record.from_peer}, created_at=${record.created_at}`)
        console.log(`       isActive=${isActive}`)
        
        if (isActive) {
          console.log(`    ✅ [DEBUG] Активное соединение (from_peer): peer_id=${record.peer_id}, from_peer=${record.from_peer}, created_at=${record.created_at}`)
          activeConnections.push({
            peer_id: record.from_peer,  // Показываем from_peer как Peer
            from_peer: record.from_peer,
            from_name: record.from_name || '-',
            from_ip: record.ip || '-',
            uuid: record.uuid || '-',
            created_at: record.created_at || '-'
          })
        }
      }
    }
    
    // Если не нашли по from_peer, ищем по peer_id
    if (activeConnections.length === 0) {
      console.log(`  🔍 [DEBUG] Не найдено по from_peer, ищем по peer_id...`)
      response = await listAudit({ 
        page: 1, 
        page_size: 50,
        peer_id: peerId
      })
      
      if (response.code === 0 && response.data && response.data.list) {
        console.log(`  📋 [DEBUG] Найдено ${response.data.list.length} записей по peer_id`)
        
        for (const record of response.data.list) {
          const isActive = (record.action === 'new' && (record.close_time === 0 || record.close_time === '0'))
          
          console.log(`    📊 record: action=${record.action}, close_time=${record.close_time}, peer_id=${record.peer_id}, from_peer=${record.from_peer}, created_at=${record.created_at}`)
          console.log(`       isActive=${isActive}`)
          
          if (isActive) {
            console.log(`    ✅ [DEBUG] Активное соединение (peer_id): peer_id=${record.peer_id}, from_peer=${record.from_peer}, created_at=${record.created_at}`)
            activeConnections.push({
              peer_id: record.peer_id,
              from_peer: record.from_peer,
              from_name: record.from_name || '-',
              from_ip: record.ip || '-',
              uuid: record.uuid || '-',
              created_at: record.created_at || '-'
            })
          }
        }
      }
    }
    
    console.log(`  ✅ [DEBUG] Всего активных соединений для peer_id=${peerId}: ${activeConnections.length}`)
    auditCache.value.set(peerId, activeConnections)
    return activeConnections
  } catch (error) {
    console.error(`  ❌ [DEBUG] Ошибка получения аудита для peer_id=${peerId}:`, error)
  }
  return []
}

// Сопоставление соединения из usage с активным аудитом
const matchWithAudit = (ip, secondsAgo, possiblePeers, currentTime) => {
  const expectedTime = currentTime - secondsAgo * 1000
  const TIME_TOLERANCE = 30000 // 30 секунд
  
  console.log(`\n🔍 [DEBUG] Сопоставление для IP=${ip}, secondsAgo=${secondsAgo}`)
  console.log(`  ⏰ expectedTime=${new Date(expectedTime).toLocaleString()}`)
  console.log(`  📋 possiblePeers:`, possiblePeers)
  
  let bestMatch = null
  let minDiff = Infinity
  
  for (const peer of possiblePeers) {
    const peerId = peer.peer_id
    const activeConnections = auditCache.value.get(peerId) || []
    console.log(`  🔄 Проверка peer_id=${peerId}, активных соединений: ${activeConnections.length}`)
    
    for (const conn of activeConnections) {
      if (conn.created_at && conn.created_at !== '-') {
        const connTime = new Date(conn.created_at).getTime()
        const diff = Math.abs(connTime - expectedTime)
        
        console.log(`    📊 conn.created_at=${conn.created_at}, diff=${diff}ms`)
        
        if (diff < minDiff && diff < TIME_TOLERANCE) {
          minDiff = diff
          bestMatch = {
            peer_id: peerId,
            hostname: peer.hostname,
            from_peer: conn.from_peer,
            from_name: conn.from_name,
            from_ip: conn.from_ip,
            uuid: conn.uuid,
            created_at: conn.created_at
          }
          console.log(`    ✅ НАЙДЕНО! diff=${diff}ms, from_name=${conn.from_name}`)
        }
      }
    }
  }
  
  if (bestMatch) {
    console.log(`✅ [DEBUG] Совпадение найдено: peer_id=${bestMatch.peer_id}, from_name=${bestMatch.from_name}, diff=${minDiff}ms`)
  } else {
    console.log(`⚠️ [DEBUG] Совпадение НЕ найдено для IP=${ip}`)
  }
  
  return bestMatch
}

// Получение активных соединений через команду usage
const fetchUsage = async () => {
  try {
    console.log('\n🚀 [DEBUG] Начало получения данных usage...')
    const currentTime = await getServerTime()
    console.log(`⏰ Текущее время сервера: ${new Date(currentTime).toLocaleString()}`)
    
    const res = await sendCmd({ cmd: 'u', target: RELAY_TARGET })
    
    if (res && res.data) {
      const lines = res.data.split('\n').filter(line => line.trim())
      console.log(`📊 [DEBUG] Получено ${lines.length} строк usage`)
      
      const connections = []
      
      for (let idx = 0; idx < lines.length; idx++) {
        const line = lines[idx]
        console.log(`\n--- Строка ${idx + 1}: ${line} ---`)
        
        const parts = line.trim().split(/\s+/)
        
        let fullId = parts[0] || '-'
        let cleanIp = fullId.replace(/^::ffff:/, '')
        cleanIp = cleanIp.replace(/:$/, '')
        
        const lastColon = cleanIp.lastIndexOf(':')
        let ip = cleanIp
        let port = ''
        
        if (lastColon !== -1) {
          ip = cleanIp.substring(0, lastColon)
          port = cleanIp.substring(lastColon + 1)
        }
        
        console.log(`📡 IP=${ip}, порт=${port}`)
        
        let timeStr = parts[1] || '0'
        const secondsAgo = parseInt(timeStr.replace(/s$/, ''))
        let timeDisplay = secondsAgo + ' сек'
        console.log(`⏱️ Время: ${secondsAgo} сек`)
        
        let total = parts[2] || '0'
        const totalValue = parseFloat(total)
        const totalUnit = total.includes('MB') ? 'MB' : (total.includes('KB') ? 'KB' : 'B')
        console.log(`📦 Объём: ${totalValue} ${totalUnit}`)
        console.log(`⚡ Скорости: max=${parts[3]}, avg=${parts[4]}, curr=${parts[5]}`)
        
        const possiblePeers = peersByExternalIp.value.get(ip) || []
        console.log(`👥 Найдено возможных пиров для IP ${ip}: ${possiblePeers.length}`)
        
        let peerId = '-'
        let hostname = '-'
        let fromPeer = '-'
        let fromName = '-'
        let fromIp = '-'
        let uuid = '-'
        let createdAt = '-'
        
        if (possiblePeers.length > 0) {
          // Загружаем активные аудиты для всех возможных пиров
          for (const peer of possiblePeers) {
            await fetchActiveAuditByPeerId(peer.peer_id)
          }
          
          // Сопоставляем по времени
          const match = matchWithAudit(ip, secondsAgo, possiblePeers, currentTime)
          
          if (match) {
            peerId = match.peer_id
            hostname = match.hostname
            fromPeer = match.from_peer
            fromName = match.from_name
            fromIp = match.from_ip
            uuid = match.uuid
            createdAt = match.created_at
            console.log(`✅ Сопоставлено: peer_id=${peerId}, from_name=${fromName}`)
          } else if (possiblePeers.length === 1) {
            peerId = possiblePeers[0].peer_id
            hostname = possiblePeers[0].hostname
            console.log(`⚠️ Используем единственного пира: peer_id=${peerId}`)
          }
        }
        
        connections.push({
          full_id: fullId,
          ip: ip,
          port: port,
          peer_id: peerId,
          hostname: hostname,
          from_peer: fromPeer,
          from_name: fromName,
          from_ip: fromIp,
          uuid: uuid,
          created_at: createdAt,
          time: timeDisplay,
          total: totalValue,
          total_unit: totalUnit,
          max_speed: formatSpeed(parts[3]),
          avg_speed: formatSpeed(parts[4]),
          current_speed: formatSpeed(parts[5])
        })
      }
      displayList.value = connections
      console.log(`\n✅ [DEBUG] Итого обработано ${connections.length} соединений\n`)
    }
  } catch (error) {
    console.error('❌ [DEBUG] Ошибка получения usage:', error)
    ElMessage.error(T('DataLoadError'))
  }
}

const getList = async () => {
  console.log('\n🔄 [DEBUG] ===== НАЧАЛО ОБНОВЛЕНИЯ ДАННЫХ =====')
  loading.value = true
  try {
    auditCache.value.clear()
    await fetchPeersList()
    await fetchUsage()
    ElMessage.success(T('DataUpdated'))
    console.log('✅ [DEBUG] ===== ОБНОВЛЕНИЕ ЗАВЕРШЕНО =====\n')
  } catch (error) {
    console.error('❌ [DEBUG] Ошибка обновления:', error)
    ElMessage.error(T('DataLoadError'))
  } finally {
    loading.value = false
  }
}

// Настройка колонок
const showColumnSetting = () => {
  columnSettingVisible.value = true
}

const saveColumnSetting = () => {
  localStorage.setItem('usage_visible_columns', JSON.stringify(allColumns.value))
  visibleColumns.value = [...allColumns.value]
  columnSettingVisible.value = false
  ElMessage.success(T('OperationSuccess'))
}

const upColumn = (index) => {
  if (index === 0) return
  const col = allColumns.value[index]
  allColumns.value.splice(index, 1)
  allColumns.value.splice(index - 1, 0, col)
}

const downColumn = (index) => {
  if (index === allColumns.value.length - 1) return
  const col = allColumns.value[index]
  allColumns.value.splice(index, 1)
  allColumns.value.splice(index + 1, 0, col)
}

const loadColumnSettings = () => {
  const saved = localStorage.getItem('usage_visible_columns')
  if (saved) {
    try {
      const parsed = JSON.parse(saved)
      if (parsed && parsed.length) {
        allColumns.value = parsed
      }
    } catch (e) {
      console.error('Error loading column settings:', e)
    }
  }
  visibleColumns.value = [...allColumns.value]
}

onMounted(() => {
  loadColumnSettings()
  getList()
})
</script>

<style scoped lang="scss">
.usage-container {
  padding: 20px;
}

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

.header-actions {
  display: flex;
  gap: 10px;
}
</style>
