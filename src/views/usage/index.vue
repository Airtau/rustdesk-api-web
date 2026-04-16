<template>
  <el-card class="usage-card" shadow="hover" v-loading="form.loading">
    <template #header>
      <div class="card-header">
        <span>{{ T('ActiveConnections') }}</span>
        <el-button type="primary" size="small" @click="getList">
          {{ T('Refresh') }}
        </el-button>
      </div>
    </template>
    <el-form :disabled="!canSend">
      <el-form-item>
        <el-table :data="form.list" size="small" border stripe>
          <el-table-column prop="0" :label="T('IP')" min-width="120" />
          <el-table-column prop="1" :label="T('Time')" min-width="160" />
          <el-table-column prop="2" :label="T('Total')" min-width="100" />
          <el-table-column prop="3" :label="T('MaxSpeed')" min-width="100" />
          <el-table-column prop="4" :label="T('AvgSpeed')" min-width="100" />
          <el-table-column prop="5" :label="T('CurrentSpeed')" min-width="120" />
        </el-table>
      </el-form-item>
    </el-form>
  </el-card>
</template>

<script setup>
import { T } from '@/utils/i18n'
import { reactive, watch, onMounted } from 'vue'
import { sendCmd } from '@/api/rustdesk'
import { RELAY_TARGET } from '@/views/rustdesk/options'

const props = defineProps({
  canSend: {
    type: Boolean,
    default: true
  }
})

const form = reactive({
  get_cmd: 'u',
  list: [],
  target: RELAY_TARGET,
  loading: false,
})

const getList = async () => {
  form.loading = true
  const res = await sendCmd({ cmd: form.get_cmd, target: RELAY_TARGET }).catch(_ => false)
  form.loading = false
  if (res && res.data) {
    form.list = res.data.split('\n').filter(i => i).map(i => i.split(/\s+/))
  }
}

watch(() => props.canSend, (v) => {
  if (v) {
    getList()
  }
}, { immediate: true })

onMounted(() => {
  if (props.canSend) {
    getList()
  }
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