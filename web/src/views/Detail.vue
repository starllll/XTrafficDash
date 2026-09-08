<template>
  <div class="container">
    <button class="back-button" @click="backToHome">
      ← 返回主页
    </button>

    <div class="header">
      <h1>
        {{ selectedService?.custom_name || selectedService?.ip }}
        <button 
          class="edit-icon" 
          @click="startEditServiceName"
          title="编辑节点名称"
        >
          ✏️
        </button>
      </h1>
    </div>

    <div class="detail-container" v-if="selectedService">
      <div class="detail-header">
        <div class="detail-title">服务信息</div>
        <button class="refresh-button" @click="refreshDetail" :disabled="isRefreshing">
          {{ isRefreshing ? '刷新中...' : '刷新数据' }}
        </button>
      </div>


      <div class="chart-section">
        <div class="chart-header">
          <div class="section-title">历史流量趋势</div>
          <div class="chart-controls">
            <button class="chart-btn" :class="{ active: quickRange === 'today' }" @click="setQuickRange('today')">今日</button>
            <button class="chart-btn" :class="{ active: quickRange === 'yesterday' }" @click="setQuickRange('yesterday')">昨日</button>
            <button class="chart-btn" :class="{ active: quickRange === 'week' }" @click="setQuickRange('week')">本周</button>
            <button class="chart-btn" :class="{ active: quickRange === 'month' }" @click="setQuickRange('month')">本月</button>
            <button class="chart-btn" :class="{ active: quickRange === '30d' }" @click="setQuickRange('30d')">近30天</button>
          </div>
        </div>
        <div class="date-range-toolbar">
          <label>
            开始
            <input type="date" v-model="dateRange.startDate" />
          </label>
          <label>
            结束
            <input type="date" v-model="dateRange.endDate" />
          </label>
          <button class="apply-btn" @click="applyCustomRange">查询</button>
        </div>
        <div class="chart-container">
          <canvas id="detail-chart"></canvas>
        </div>
      </div>

      <div class="traffic-tables">
        <div class="traffic-table">
          <div class="table-title">入站今日流量</div>
          <div 
            v-for="inbound in sortedInbounds" 
            :key="inbound.id" 
            class="table-row"
            @click="viewPortDetail(selectedService.id, inbound.tag)"
            style="cursor:pointer;"
          >
            <div class="table-label-container">
              <div 
                class="table-label"
              >
                <span class="display-name">
                  {{ inbound.custom_name || inbound.tag }}
                  <button 
                    class="edit-icon" 
                    @click.stop="startEditInbound(inbound)"
                    title="编辑入站名称"
                  >
                    ✏️
                  </button>
                </span>
              </div>
            </div>
            <div class="table-value">
              <span class="upload-traffic">
                <span class="traffic-icon">↑</span>
                {{ formatBytes(inbound.up) }}
              </span>
              <span class="download-traffic">
                <span class="traffic-icon">↓</span>
                {{ formatBytes(inbound.down) }}
              </span>
            </div>
          </div>
        </div>

        <div class="traffic-table">
          <div class="table-title">用户今日流量</div>
          <div 
            v-for="client in sortedClients" 
            :key="client.id" 
            class="table-row"
            @click="viewUserDetail(selectedService.id, client.email)"
            style="cursor:pointer;"
          >
            <div class="table-label-container">
              <div 
                class="table-label"
              >
                <span class="display-name">
                  {{ client.custom_name || client.email }}
                  <button 
                    class="edit-icon" 
                    @click.stop="startEditClient(client)"
                    title="编辑用户名称"
                  >
                    ✏️
                  </button>
                </span>
              </div>
            </div>
            <div class="table-value">
              <span class="upload-traffic">
                <span class="traffic-icon">↑</span>
                {{ formatBytes(client.up) }}
              </span>
              <span class="download-traffic">
                <span class="traffic-icon">↓</span>
                {{ formatBytes(client.down) }}
              </span>
            </div>
          </div>
        </div>
      </div>

      
      
    </div>
  </div>
  
  <!-- 编辑节点名称弹窗 -->
  <EditNameModal
    v-model:visible="showServiceModal"
    :value="currentEditingValue"
    title="编辑节点名称"
    label="节点名称"
    placeholder="请输入节点名称"
    @save="saveServiceName"
    @close="closeServiceModal"
  />
  
  <!-- 编辑入站名称弹窗 -->
  <EditNameModal
    v-model:visible="showInboundModal"
    :value="currentEditingValue"
    title="编辑入站名称"
    label="入站名称"
    placeholder="请输入入站名称"
    @save="saveInboundName"
    @close="closeInboundModal"
  />
  
  <!-- 编辑用户名称弹窗 -->
  <EditNameModal
    v-model:visible="showClientModal"
    :value="currentEditingValue"
    title="编辑用户名称"
    label="用户名称"
    placeholder="请输入用户名称"
    @save="saveClientName"
    @close="closeClientModal"
  />
</template>

<script setup>
import { onMounted, onUnmounted, computed, ref } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { useServicesStore } from '../stores/services'
import { formatBytes as rawFormatBytes, formatDate, formatDateTime } from '../utils/formatters'
import { servicesAPI } from '../utils/api'
import Chart from 'chart.js/auto'
import EditNameModal from '../components/EditNameModal.vue'

const route = useRoute()
const router = useRouter()
const servicesStore = useServicesStore()

const selectedService = computed(() => servicesStore.selectedService)

let detailChart = null
let refreshInterval = null
const isRefreshing = ref(false)
const quickRange = ref('today')
const dateRange = ref({ startDate: '', endDate: '' })

const formatInputDate = (date) => {
  const year = date.getFullYear()
  const month = String(date.getMonth() + 1).padStart(2, '0')
  const day = String(date.getDate()).padStart(2, '0')
  return `${year}-${month}-${day}`
}

const setQuickRange = async (key) => {
  const now = new Date()
  const start = new Date(now)
  const end = new Date(now)
  quickRange.value = key

  if (key === 'today') {
    start.setHours(0, 0, 0, 0)
    end.setHours(23, 59, 59, 999)
  } else if (key === 'yesterday') {
    start.setDate(start.getDate() - 1)
    end.setDate(end.getDate() - 1)
    start.setHours(0, 0, 0, 0)
    end.setHours(23, 59, 59, 999)
  } else if (key === 'week') {
    const day = start.getDay() || 7
    start.setDate(start.getDate() - day + 1)
    start.setHours(0, 0, 0, 0)
    end.setHours(23, 59, 59, 999)
  } else if (key === 'month') {
    start.setDate(1)
    start.setHours(0, 0, 0, 0)
    end.setHours(23, 59, 59, 999)
  } else if (key === '30d') {
    start.setDate(start.getDate() - 29)
    start.setHours(0, 0, 0, 0)
    end.setHours(23, 59, 59, 999)
  }

  dateRange.value = { startDate: formatInputDate(start), endDate: formatInputDate(end) }
  await createDetailChart({ startDate: dateRange.value.startDate, endDate: dateRange.value.endDate })
}

const applyCustomRange = async () => {
  if (!dateRange.value.startDate || !dateRange.value.endDate) {
    alert('请选择开始和结束日期')
    return
  }
  quickRange.value = 'custom'
  await createDetailChart(dateRange.value)
}

// 弹窗相关状态
const showServiceModal = ref(false)
const showInboundModal = ref(false)
const showClientModal = ref(false)
const currentEditingInbound = ref(null)
const currentEditingClient = ref(null)
const currentEditingValue = ref('')

// 编辑相关函数
const startEditInbound = (inbound) => {
  currentEditingInbound.value = inbound
  currentEditingValue.value = inbound.custom_name || inbound.tag
  showInboundModal.value = true
}

const saveInboundName = async (newName) => {
  try {
    const response = await servicesAPI.updateInboundCustomName(
      selectedService.value.id, 
      currentEditingInbound.value.tag, 
      newName
    )
    if (response.data.success) {
      currentEditingInbound.value.custom_name = newName
      // 同时更新selectedService中对应的端口数据
      const inboundInService = selectedService.value.inbound_traffics.find(
        i => i.tag === currentEditingInbound.value.tag
      )
      if (inboundInService) {
        inboundInService.custom_name = newName
      }
      showInboundModal.value = false
    } else {
      alert('保存失败: ' + response.data.error)
    }
  } catch (error) {
    console.error('保存入站失败:', error)
    alert('保存失败: ' + error.message)
  }
}

const closeInboundModal = () => {
  showInboundModal.value = false
  currentEditingInbound.value = null
}

const startEditClient = (client) => {
  currentEditingClient.value = client
  currentEditingValue.value = client.custom_name || client.email
  showClientModal.value = true
}

const saveClientName = async (newName) => {
  try {
    const response = await servicesAPI.updateClientCustomName(
      selectedService.value.id, 
      currentEditingClient.value.email, 
      newName
    )
    if (response.data.success) {
      currentEditingClient.value.custom_name = newName
      // 同时更新selectedService中对应的用户数据
      const clientInService = selectedService.value.client_traffics.find(
        c => c.email === currentEditingClient.value.email
      )
      if (clientInService) {
        clientInService.custom_name = newName
      }
      showClientModal.value = false
    } else {
      alert('保存失败: ' + response.data.error)
    }
  } catch (error) {
    console.error('保存用户名称失败:', error)
    alert('保存失败: ' + error.message)
  }
}

const closeClientModal = () => {
  showClientModal.value = false
  currentEditingClient.value = null
}

// 编辑节点名称
const startEditServiceName = () => {
  currentEditingValue.value = selectedService.value?.custom_name || selectedService.value?.ip
  showServiceModal.value = true
}

const saveServiceName = async (newName) => {
  try {
    const response = await servicesAPI.updateServiceCustomName(selectedService.value.id, newName)
    if (response.data.success) {
      selectedService.value.custom_name = newName
      showServiceModal.value = false
    } else {
      alert('保存失败: ' + response.data.error)
    }
  } catch (error) {
    console.error('保存节点名称失败:', error)
    alert('保存失败: ' + error.message)
  }
}

const closeServiceModal = () => {
  showServiceModal.value = false
}

const createDetailChart = async (range = {}) => {
  try {
    const response = await servicesAPI.getServiceTrafficHistory(selectedService.value.id, range)
    if (response.data.success) {
      const data = response.data.data
      const ctx = document.getElementById('detail-chart')
      
      if (ctx) {
        // 销毁现有图表
        if (detailChart) {
          detailChart.destroy()
        }
        
        detailChart = new Chart(ctx, {
          type: 'line',
          data: {
            labels: data.dates.map(date => data.granularity === 'day' ? formatDate(date) : formatDateTime(date)),
            datasets: [
              {
                label: '上传',
                data: data.upload_data,
                borderColor: '#74b9ff',
                backgroundColor: 'rgba(116, 185, 255, 0.1)',
                tension: 0.4,
                fill: true
              },
              {
                label: '下载',
                data: data.download_data,
                borderColor: '#00b894',
                backgroundColor: 'rgba(0, 184, 148, 0.1)',
                tension: 0.4,
                fill: true
              }
            ]
          },
          options: {
            responsive: true,
            maintainAspectRatio: false,
            plugins: {
              legend: {
                display: true,
                position: 'top',
                labels: {
                  color: '#2c3e50',
                  font: {
                    size: 14,
                    weight: 'bold'
                  }
                }
              }
            },
            scales: {
              x: {
                display: true,
                title: {
                  display: true,
                  text: data.granularity === 'day' ? '日期' : '时间',
                  color: '#2c3e50',
                  font: {
                    size: 14,
                    weight: 'bold'
                  }
                },
                ticks: {
                  color: '#2c3e50',
                  font: {
                    size: 12
                  }
                },
                grid: {
                  color: 'rgba(44, 62, 80, 0.1)'
                }
              },
              y: {
                display: true,
                title: {
                  display: true,
                  text: '流量',
                  color: '#2c3e50',
                  font: {
                    size: 14,
                    weight: 'bold'
                  }
                },
                ticks: {
                  color: '#2c3e50',
                  font: {
                    size: 12
                  },
                  callback: function(value, index, values) {
                    if (value >= 1024 * 1024 * 1024) {
                      return (value / (1024 * 1024 * 1024)).toFixed(1) + ' GB';
                    } else if (value >= 1024 * 1024) {
                      return (value / (1024 * 1024)).toFixed(1) + ' MB';
                    } else if (value >= 1024) {
                      return (value / 1024).toFixed(1) + ' KB';
                    } else {
                      return value + ' B';
                    }
                  }
                },
                grid: {
                  color: 'rgba(44, 62, 80, 0.1)'
                }
              }
            }
          }
        })
      }
    }
  } catch (error) {
    console.error('创建详情图表失败:', error)
  }
}

const refreshDetail = async () => {
  if (selectedService.value && !isRefreshing.value) {
    isRefreshing.value = true
    try {
      await servicesStore.loadServiceDetail(selectedService.value.id)
      await createDetailChart(dateRange.value)
    } catch (error) {
      console.error('刷新数据失败:', error)
    } finally {
      isRefreshing.value = false
    }
  }
}

// 开始自动刷新
const startAutoRefresh = () => {
  if (refreshInterval) {
    clearInterval(refreshInterval)
  }
  // 每60秒刷新一次，确保图表数据实时更新
  refreshInterval = setInterval(async () => {
    if (selectedService.value) {
      await servicesStore.loadServiceDetail(selectedService.value.id)
      await createDetailChart(dateRange.value)
    }
  }, 60000)
}

// 停止自动刷新
const stopAutoRefresh = () => {
  if (refreshInterval) {
    clearInterval(refreshInterval)
    refreshInterval = null
  }
}

const backToHome = () => {
  router.push('/home')
}

const viewPortDetail = (serviceId, tag) => {
  router.push(`/port/${serviceId}/${tag}`)
}

const viewUserDetail = (serviceId, email) => {
  router.push(`/user/${serviceId}/${email}`)
}

// 保留4位有效数字的格式化函数
function formatBytes(num) {
  if (typeof num !== 'number' || isNaN(num)) return '-';
  if (num >= 1024 * 1024 * 1024) {
    return (num / (1024 * 1024 * 1024)).toPrecision(4) + ' GB';
  } else if (num >= 1024 * 1024) {
    return (num / (1024 * 1024)).toPrecision(4) + ' MB';
  } else if (num >= 1024) {
    return (num / 1024).toPrecision(4) + ' KB';
  } else {
    return num + ' B';
  }
}

const sortedInbounds = computed(() => {
  if (!selectedService.value || !Array.isArray(selectedService.value.inbound_traffics)) return [];
  return [...selectedService.value.inbound_traffics].sort((a, b) => b.down - a.down);
});
const sortedClients = computed(() => {
  if (!selectedService.value || !Array.isArray(selectedService.value.client_traffics)) return [];
  return [...selectedService.value.client_traffics].sort((a, b) => b.down - a.down);
});

onMounted(async () => {
  const serviceId = parseInt(route.params.serviceId)
  
  // 如果没有选中的服务，从主页获取
  if (!selectedService.value || selectedService.value.id !== serviceId) {
    // 这里需要从主页获取服务列表，然后选择对应的服务
    await servicesStore.loadServices()
    const service = servicesStore.services.find(s => s.id === serviceId)
    if (service) {
      servicesStore.selectService(service)
    }
  }
  
  // 加载服务详情
  if (selectedService.value) {
    await servicesStore.loadServiceDetail(serviceId)
    await createDetailChart()
    startAutoRefresh()
  }
})

onUnmounted(() => {
  if (detailChart) {
    detailChart.destroy()
  }
  stopAutoRefresh()
})
</script>

<style scoped>
.date-range-toolbar {
  display: flex;
  flex-wrap: wrap;
  align-items: end;
  gap: 12px;
  margin: 16px 0 20px;
}

.date-range-toolbar label {
  display: flex;
  flex-direction: column;
  gap: 6px;
  font-size: 0.85rem;
  color: #495057;
}

.date-range-toolbar input {
  min-width: 150px;
  padding: 8px 10px;
  border: 1px solid #dfe6ee;
  border-radius: 8px;
  background: #fff;
  color: #2c3e50;
}

.apply-btn {
  padding: 9px 16px;
  border: none;
  border-radius: 8px;
  background: #70A1FF;
  color: white;
  font-weight: 600;
  cursor: pointer;
}

.table-label-container {
  display: flex;
  align-items: center;
  gap: 8px;
}

.display-name {
  display: flex;
  align-items: center;
  gap: 4px;
  font-weight: 500;
  color: #2c3e50;
}

.edit-icon {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 12px;
  padding: 2px;
  border-radius: 3px;
  transition: all 0.2s ease;
}

.edit-icon:hover {
  background: rgba(52, 152, 219, 0.1);
}

/* 标题中的编辑图标样式 */
.header h1 .edit-icon {
  font-size: 16px;
  padding: 4px;
  margin-left: 8px;
  vertical-align: middle;
}

.header h1 {
  color: #222;
  text-shadow: none;
}
.detail-title {
  color: #222;
}
.section-title {
  color: #222;
}

.chart-section {
  background: white;
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 4px 15px rgba(0,0,0,0.1);
  margin-top: 25px;
  margin-bottom: 25px;
}

.chart-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.section-title {
  font-size: 1.2rem;
  font-weight: bold;
  color: #2c3e50;
  margin: 0;
}

.chart-controls {
  display: flex;
  gap: 8px;
}

.chart-btn {
  padding: 6px 12px;
  border: 1.5px solid #70A1FF;
  background: #fff;
  color: #70A1FF;
  border-radius: 20px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 500;
  transition: background 0.2s, color 0.2s, border-color 0.2s, box-shadow 0.2s, transform 0.18s;
  margin-right: 8px;
}
.chart-btn:last-child { margin-right: 0; }

.chart-btn:hover {
  background: #EAF3FF;
  color: #1E90FF;
  border-color: #1E90FF;
  transform: translateY(-1px);
  box-shadow: 0 2px 8px rgba(112,161,255,0.10);
}

.chart-btn.active {
  background: #70A1FF;
  color: #fff;
  border-color: #70A1FF;
  box-shadow: 0 2px 8px rgba(112,161,255,0.18);
}

.chart-container {
  height: 400px;
  position: relative;
}

.refresh-button {
  background: #70A1FF;
  color: #fff;
  border: none;
  padding: 10px 20px;
  border-radius: 20px;
  cursor: pointer;
  font-size: 0.9rem;
  font-weight: 600;
  box-shadow: 0 2px 8px rgba(112,161,255,0.10);
  transition: background 0.2s, color 0.2s, box-shadow 0.2s, transform 0.18s;
  position: relative;
  overflow: hidden;
}

.refresh-button:hover:not(:disabled) {
  background: #1E90FF;
  color: #fff;
  transform: translateY(-1px);
  box-shadow: 0 4px 16px rgba(112,161,255,0.18);
}

.refresh-button:disabled {
  background: #d1d1d6;
  color: #fff;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

.table-row {
  border-left: 3px solid #70A1FF;
}

.container {
  max-width: 900px;
  margin: 0 auto;
  padding: 0 24px;
}
</style> 