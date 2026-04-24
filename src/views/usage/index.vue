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
            v-if="col.name === 'port'"
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
          >
            <template #default="{ row }">
              {{ formatDuration(row.time_seconds) }}
            </template>
          </el-table-column>
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

// Форматирование длительности
const formatDuration = (seconds) => {
  if (!seconds || seconds === 0) return '0 сек'
  
  const days = Math.floor(seconds / 86400)
  const hours = Math.floor((seconds % 86400) / 3600)
  const minutes = Math.floor((seconds % 3600) / 60)
  const secs = seconds % 60
  
  const parts = []
  if (days > 0) parts.push(`${days} дн`)
  if (hours > 0) parts.push(`${hours} ч`)
  if (minutes > 0) parts.push(`${minutes} мин`)
  if (secs > 0 || parts.length === 0) parts.push(`${secs} сек`)
  
  return parts.join(' ')
}

// Все доступные колонки (без колонки ip)
const allColumns = ref([
  { name: 'created_at', visible: true, label: 'CreatedAt', width: 150 },
  { name: 'target_ip', visible: true, label: 'IP', width: 100 },
  { name: 'port', visible: true, label: 'Port', width: 60 },
  { name: 'hostname', visible: true, label: 'Hostname', width: 120 },
  { name: 'peer_id', visible: true, label: 'Peer', width: 90 },
  { name: 'from_peer', visible: true, label: 'FromPeer', width: 90 },
  { name: 'from_name', visible: true, label: 'FromName', width: 150 },
  { name: 'from_ip', visible: true, label: 'FromIP', width: 100 },
  { name: 'uuid', visible: false, label: 'Uuid', width: 300 },
  { name: 'time', visible: true, label: 'Time', width: 150 },
  { name: 'total', visible: true, label: 'Total', width: 90 },
  { name: 'max_speed', visible: true, label: 'MaxSpeed', width: 90 },
  { name: 'avg_speed', visible: true, label: 'AvgSpeed', width: 90 },
  { name: 'current_speed', visible: true, label: 'CurrentSpeed', width: 90 }
])

const visibleColumns = ref([])
const activeAuditCache = ref([])

const formatSpeed = (speedStr) => {
  if (!speedStr) return 0
  return parseFloat(speedStr)
}

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

const fetchActiveAudit = async () => {
  try {
    const response = await listActive({
      page: 1,
      page_size: 100
    })

    if (response.code === 0 && response.data && response.data.list) {
      activeAuditCache.value = response.data.list
      return response.data.list
    }
    return []
  } catch (error) {
    console.error('Error fetching active connections:', error)
    return []
  }
}

const fetchUsage = async () => {
  try {
    const currentTime = await getServerTime()
    const res = await sendCmd({ cmd: 'u', target: RELAY_TARGET })
    const activeAudit = await fetchActiveAudit()

    if (res && res.data) {
      const lines = res.data.split('\n').filter(line => line.trim())
      const connections = []

      for (const line of lines) {
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

        let timeStr = parts[1] || '0'
        const secondsAgo = parseInt(timeStr.replace(/s$/, ''))
        
        const expectedTime = currentTime - secondsAgo * 1000
        const TIME_TOLERANCE = 30000

        let total = parts[2] || '0'
        const totalValue = parseFloat(total)
        const totalUnit = total.includes('MB') ? 'MB' : (total.includes('KB') ? 'KB' : 'B')

        let match = null
        let minDiff = Infinity

        for (const audit of activeAudit) {
          const ipMatches = (audit.from_ip === ip) || (audit.target_ip === ip)
          
          if (ipMatches && audit.created_at && audit.created_at !== '-') {
            const auditTime = new Date(audit.created_at).getTime()
            const diff = Math.abs(auditTime - expectedTime)
            
            if (diff < minDiff && diff < TIME_TOLERANCE) {
              minDiff = diff
              match = audit
            }
          }
        }

        connections.push({
          port: port,
          target_ip: match?.target_ip || '-',
          peer_id: match?.peer_id || '-',
          hostname: match?.hostname || '-',
          from_peer: match?.from_peer || '-',
          from_name: match?.from_name || '-',
          from_ip: match?.from_ip || '-',
          uuid: match?.uuid || '-',
          created_at: match?.created_at || '-',
          time_seconds: secondsAgo,
          time: secondsAgo + ' сек',
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
    await fetchUsage()
    ElMessage.success(T('DataUpdated'))
  } catch (error) {
    console.error('Update error:', error)
    ElMessage.error(T('DataLoadError'))
  } finally {
    loading.value = false
  }
}

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
