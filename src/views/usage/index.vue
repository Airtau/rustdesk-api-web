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
            v-else-if="col.name === 'target_ip'"
            prop="target_ip"
            :label="T('TargetIP')"
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
import { listActive } from '@/api/audit'

const loading = ref(false)
const displayList = ref([])
const columnSettingVisible = ref(false)

// Все доступные колонки
const allColumns = ref([
  { name: 'ip', visible: true, label: 'IP', width: 120 },
  { name: 'port', visible: true, label: 'Port', width: 80 },
  { name: 'target_ip', visible: true, label: 'TargetIP', width: 120 },
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

// Кэш активных соединений из аудита (с данными из peers)
const activeAuditCache = ref([])

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
      const serverTime = new Date(date).getTime()
      console.log(`🕐 [DEBUG] Server time from header: ${new Date(serverTime).toLocaleString()}`)
      return serverTime
    }
  } catch (error) {
    console.error('Error getting server time:', error)
  }
  const localTime = Date.now()
  console.log(`🕐 [DEBUG] Using local time: ${new Date(localTime).toLocaleString()}`)
  return localTime
}

// Получение всех активных соединений из нового эндпоинта
const fetchActiveAudit = async () => {
  try {
    const response = await listActive({
      page: 1,
      page_size: 100
    })

    if (response.code === 0 && response.data && response.data.list) {
      console.log(`✅ [DEBUG] Всего активных соединений из API: ${response.data.list.length}`)
      console.log(`📋 [DEBUG] API данные:`, response.data.list.map(item => ({
        id: item.id,
        from_ip: item.from_ip,
        created_at: item.created_at,
        peer_id: item.peer_id,
        hostname: item.hostname
      })))
      activeAuditCache.value = response.data.list
      return response.data.list
    }
    return []
  } catch (error) {
    console.error('Error fetching active connections:', error)
    return []
  }
}

// Получение активных соединений через команду usage и сопоставление с audit
const fetchUsage = async () => {
  try {
    const currentTime = await getServerTime()
    console.log(`🕐 [DEBUG] Current timestamp: ${currentTime}`)

    const res = await sendCmd({ cmd: 'u', target: RELAY_TARGET })

    // Получаем активные соединения из аудита
    const activeAudit = await fetchActiveAudit()

    if (res && res.data) {
      const lines = res.data.split('\n').filter(line => line.trim())
      console.log(`📡 [DEBUG] Usage command output:\n${res.data}`)
      console.log(`📊 [DEBUG] Total usage lines: ${lines.length}`)

      const connections = []

      for (const line of lines) {
        const parts = line.trim().split(/\s+/)

        // Парсим IP и порт из usage
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

        // Время в секундах
        let timeStr = parts[1] || '0'
        const secondsAgo = parseInt(timeStr.replace(/s$/, ''))
        let timeDisplay = secondsAgo + ' сек'

        // Вычисляем ожидаемое время начала
        const expectedTime = currentTime - secondsAgo * 1000
        const expectedTimeStr = new Date(expectedTime).toLocaleString()

        console.log(`\n🔍 [DEBUG] === Анализ соединения ===`)
        console.log(`  📍 IP:Port: ${ip}:${port}`)
        console.log(`  ⏱️  secondsAgo: ${secondsAgo} сек (${Math.floor(secondsAgo/60)} мин ${secondsAgo%60} сек)`)
        console.log(`  🕐 Current server time: ${new Date(currentTime).toLocaleString()}`)
        console.log(`  ⏰ Expected start time: ${expectedTimeStr} (timestamp: ${expectedTime})`)

        // Объём данных
        let total = parts[2] || '0'
        const totalValue = parseFloat(total)
        const totalUnit = total.includes('MB') ? 'MB' : (total.includes('KB') ? 'KB' : 'B')

        const TIME_TOLERANCE = 30000 // Увеличил до 30 секунд для лучшего сопоставления
        console.log(`  ⚙️  Time tolerance: ${TIME_TOLERANCE} ms (${TIME_TOLERANCE/1000} sec)`)

	// Сопоставляем с активными соединениями из аудита по from_ip и времени
	let match = null
	let minDiff = Infinity
	let bestMatchTime = null

	const matchingByIp = activeAudit.filter(a => (a.from_ip === ip) || (a.target_ip === ip))
	console.log(`  🔎 Found ${matchingByIp.length} API records with IP match (from_ip=${ip} OR target_ip=${ip})`)

	for (const audit of activeAudit) {
	  // Проверяем совпадение по from_ip ИЛИ target_ip
	  const ipMatches = (audit.from_ip === ip) || (audit.target_ip === ip);

	  if (ipMatches) {
	    if (audit.created_at && audit.created_at !== '-') {
	      const auditTime = new Date(audit.created_at).getTime()
	      const diff = Math.abs(auditTime - expectedTime)
	      const diffSeconds = Math.floor(diff / 1000)

	      console.log(`    📝 Comparing with API record:`)
	      console.log(`      - created_at: ${audit.created_at}`)
	      console.log(`      - from_ip: ${audit.from_ip}, target_ip: ${audit.target_ip}`)
	      console.log(`      - audit timestamp: ${auditTime}`)
	      console.log(`      - diff: ${diff} ms (${diffSeconds} sec)`)
	      console.log(`      - within tolerance: ${diff < TIME_TOLERANCE ? '✅ YES' : '❌ NO'}`)

	      if (diff < minDiff && diff < TIME_TOLERANCE) {
		minDiff = diff
		match = audit
		bestMatchTime = audit.created_at
		console.log(`      ✨ NEW BEST MATCH! diff=${diff}ms`)
	      }
	    } else {
	      console.log(`    ⚠️ API record has empty created_at, skipping`)
	    }
	  }
	}

	if (match) {
	  const matchDiffSeconds = Math.floor(minDiff / 1000)
	  console.log(`  ✅ MATCH FOUND!`)
	  console.log(`    - API created_at: ${bestMatchTime}`)
	  console.log(`    - Expected time: ${expectedTimeStr}`)
	  console.log(`    - Difference: ${minDiff} ms (${matchDiffSeconds} sec)`)
	  console.log(`    - Peer ID: ${match.peer_id}`)
	  console.log(`    - Hostname: ${match.hostname}`)
	  console.log(`    - Match type: ${match.from_ip === ip ? 'from_ip' : 'target_ip'}`)
	} else {
	  console.log(`  ❌ NO MATCH FOUND for IP=${ip}, time=${secondsAgo}s`)
	  if (matchingByIp.length === 0) {
	    console.log(`    - No API records with from_ip=${ip} or target_ip=${ip}`)
	  } else {
	    console.log(`    - Found ${matchingByIp.length} records with matching IP, but time mismatch > ${TIME_TOLERANCE/1000} sec`)
	  }
	}

        connections.push({
          full_id: fullId,
          ip: ip,
          port: port,
          target_ip: match?.target_ip || '-',
          peer_id: match?.peer_id || '-',
          hostname: match?.hostname || '-',
          from_peer: match?.from_peer || '-',
          from_name: match?.from_name || '-',
          from_ip: match?.from_ip || '-',
          uuid: match?.uuid || '-',
          created_at: match?.created_at || '-',
          time: timeDisplay,
          total: totalValue,
          total_unit: totalUnit,
          max_speed: formatSpeed(parts[3]),
          avg_speed: formatSpeed(parts[4]),
          current_speed: formatSpeed(parts[5])
        })
      }
      displayList.value = connections

      const matchedCount = connections.filter(c => c.peer_id !== '-').length
      console.log(`\n✅ [DEBUG] Total connections processed: ${connections.length}`)
      console.log(`   └─ With matches: ${matchedCount}`)
      console.log(`   └─ Without matches: ${connections.length - matchedCount}`)
    }
  } catch (error) {
    console.error('❌ [DEBUG] Error fetching usage:', error)
    ElMessage.error(T('DataLoadError'))
  }
}

const getList = async () => {
  console.log('\n🔄 [DEBUG] ===== START DATA UPDATE =====')
  loading.value = true
  try {
    await fetchUsage()
    ElMessage.success(T('DataUpdated'))
    console.log('✅ [DEBUG] ===== UPDATE COMPLETED =====\n')
  } catch (error) {
    console.error('❌ [DEBUG] Update error:', error)
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
