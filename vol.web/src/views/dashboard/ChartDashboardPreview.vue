<template>
  <div class="chart-dashboard-preview">
    <div class="preview-header">
      <h2>{{ dashboardTitle }}</h2>
      <el-button type="primary" size="small" @click="closePreview">关闭预览</el-button>
    </div>
    <div class="preview-content">
      <div 
        v-for="(item, index) in dashboardItems" 
        :key="item.id || index"
        class="dashboard-item"
        :style="{
          width: item.width || '300px',
          height: item.height || '250px',
          left: item.left || `${(index % 3) * 320}px`,
          top: item.top || `${Math.floor(index / 3) * 280}px`
        }"
      >
        <div class="item-header">
          <span class="item-title">{{ item.title || item.name }}</span>
        </div>
        <div 
          class="chart-container" 
          :id="`chart-${item.id || index}`"
        ></div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, onUnmounted } from 'vue'
import * as echarts from 'echarts'
import http from '@/api/http.js'

// 工作台项目列表
const dashboardItems = ref([])
// 图表实例引用
const chartInstances = reactive({})
// 自动刷新定时器
const refreshTimers = reactive({})
// 仪表盘标题
const dashboardTitle = ref('仪表盘预览')

// 组件挂载时加载预览数据
onMounted(async () => {
  // 从sessionStorage获取预览数据
  const previewConfig = sessionStorage.getItem('chartDashboardPreview')
  
  if (previewConfig) {
    try {
      const config = JSON.parse(previewConfig)
      dashboardItems.value = config.items || []
      
      // 设置标题
      dashboardTitle.value = `仪表盘预览 - ${new Date(config.timestamp).toLocaleString()}`
      
      // 延迟渲染图表，确保DOM已更新
      setTimeout(() => {
        dashboardItems.value.forEach((item, index) => {
          renderChart(item, index)
        })
      }, 100)
    } catch (error) {
      console.error('加载预览数据失败:', error)
    }
  } else {
    // 如果没有预览数据，尝试加载默认配置
    try {
      const response = await http.post('/api/chartDashboard/getDefault', {}, false)
      if (response.status && response.data) {
        dashboardItems.value = response.data.items || []
        
        setTimeout(() => {
          dashboardItems.value.forEach((item, index) => {
            renderChart(item, index)
          })
        }, 100)
      }
    } catch (error) {
      console.error('加载默认仪表盘失败:', error)
    }
  }
})

// 渲染图表
const renderChart = async (item, index) => {
  const chartId = `chart-${item.id}`
  const chartDom = document.getElementById(chartId)
  
  if (!chartDom) return
  
  // 销毁已存在的实例
  if (chartInstances[chartId]) {
    chartInstances[chartId].dispose()
  }
  
  // 创建新实例
  const chartInstance = echarts.init(chartDom)
  chartInstances[chartId] = chartInstance
  
  // 获取图表配置
  let option = null
  
  // 尝试从API获取数据
  const apiData = await fetchChartData(item)
  if (apiData) {
    // 如果有API数据，使用API数据生成配置
    option = getChartOptionWithApiData(item, apiData)
  } else {
    // 否则使用模拟数据
    option = getChartOption(item)
  }
  
  // 设置配置项
  chartInstance.setOption(option)
  
  // 添加窗口大小改变监听
  window.addEventListener('resize', () => {
    chartInstance.resize()
  })
  
  // 设置自动刷新
  setupAutoRefresh(item)
}

// 获取图表数据
const fetchChartData = async (item) => {
  if (item.dataType === 'api' && item.apiUrl) {
    try {
      // 从API获取数据
      const response = await http.post(item.apiUrl, {}, false)
      if (response.status || response.success) {
        return response.data || response.result
      }
    } catch (error) {
      console.error('API请求失败:', error)
    }
  }
  // 返回null，使用模拟数据
  return null
}

// 获取图表配置（使用模拟数据）
const getChartOption = (item) => {
  const dateArr = generateDateArray()
  
  switch (item.type) {
    case 'line':
    case 'bar':
      return getLineBarOption(item, dateArr)
    case 'pie':
      return getPieOption(item)
    case 'scatter':
      return getScatterOption()
    case 'radar':
      return getRadarOption()
    case 'gauge':
      return getGaugeOption()
    default:
      return getLineBarOption(item, dateArr)
  }
}

// 获取图表配置（使用API数据）
const getChartOptionWithApiData = (item, apiData) => {
  if (apiData && apiData.chartOption) {
    return {
      ...apiData.chartOption,
      title: {
        ...apiData.chartOption.title,
        text: item.title || apiData.chartOption.title?.text,
        textStyle: {
          fontSize: 14
        }
      }
    }
  }
  
  const dateArr = generateDateArray()
  
  switch (item.type) {
    case 'line':
    case 'bar':
      if (apiData && apiData.xAxis && apiData.series) {
        return getLineBarOption(item, apiData.xAxis, apiData.series)
      }
      return getLineBarOption(item, dateArr)
    case 'pie':
      if (apiData && apiData.data) {
        return getPieOption(item, apiData.data)
      }
      return getPieOption(item)
    case 'scatter':
      if (apiData && apiData.data) {
        return getScatterOption(apiData.data)
      }
      return getScatterOption()
    case 'radar':
      if (apiData && apiData.indicator && apiData.data) {
        return getRadarOption(apiData.indicator, apiData.data)
      }
      return getRadarOption()
    case 'gauge':
      if (apiData && apiData.value !== undefined) {
        return getGaugeOption(apiData.value)
      }
      return getGaugeOption()
    default:
      return getLineBarOption(item, dateArr)
  }
}

// 生成日期数组
const generateDateArray = () => {
  const dates = []
  const now = new Date()
  for (let i = 29; i >= 0; i--) {
    const date = new Date(now)
    date.setDate(date.getDate() - i)
    dates.push(`${date.getMonth() + 1}-${date.getDate()}`)
  }
  return dates
}

// 获取折线图/柱状图配置
const getLineBarOption = (item, dateArr, customSeries) => {
  const series = []
  
  if (customSeries) {
    return {
      title: {
        text: item.title,
        textStyle: {
          fontSize: 14
        }
      },
      tooltip: {
        trigger: 'axis'
      },
      legend: {
        data: customSeries.map(s => s.name)
      },
      xAxis: {
        type: 'category',
        data: dateArr,
        axisLabel: {
          rotate: 45
        }
      },
      yAxis: {
        type: 'value'
      },
      series: customSeries
    }
  }
  
  if (item.subType === 'line' || item.subType === 'mix') {
    series.push({
      name: '数据一',
      type: 'line',
      smooth: true,
      data: generateRandomData(30, 100, 500)
    })
  }
  
  if (item.subType === 'bar' || item.subType === 'mix') {
    series.push({
      name: '数据二',
      type: 'bar',
      data: generateRandomData(30, 50, 300)
    })
  }
  
  return {
    title: {
      text: item.title,
      textStyle: {
        fontSize: 14
      }
    },
    tooltip: {
      trigger: 'axis'
    },
    legend: {
      data: series.map(s => s.name)
    },
    xAxis: {
      type: 'category',
      data: dateArr.slice(-7),
      axisLabel: {
        rotate: 45
      }
    },
    yAxis: {
      type: 'value'
    },
    series: series
  }
}

// 获取饼图配置
const getPieOption = (item, customData) => {
  const data = customData || [
    { value: 335, name: '类别一' },
    { value: 310, name: '类别二' },
    { value: 234, name: '类别三' },
    { value: 135, name: '类别四' },
    { value: 1548, name: '类别五' }
  ]
  
  return {
    title: {
      text: item.title,
      textStyle: {
        fontSize: 14
      }
    },
    tooltip: {
      trigger: 'item'
    },
    legend: {
      orient: 'vertical',
      left: 'left'
    },
    series: [{
      name: '访问来源',
      type: 'pie',
      radius: item.subType === 'doughnut' ? ['40%', '70%'] : '60%',
      data: data,
      emphasis: {
        itemStyle: {
          shadowBlur: 10,
          shadowOffsetX: 0,
          shadowColor: 'rgba(0, 0, 0, 0.5)'
        }
      }
    }]
  }
}

// 获取散点图配置
const getScatterOption = (customData) => {
  const data = customData || generateScatterData(50)
  
  return {
    title: {
      text: '散点图',
      textStyle: {
        fontSize: 14
      }
    },
    tooltip: {
      trigger: 'item',
      formatter: function(params) {
        return `(${params.value[0]}, ${params.value[1]})`
      }
    },
    xAxis: {
      type: 'value'
    },
    yAxis: {
      type: 'value'
    },
    series: [{
      symbolSize: 8,
      data: data,
      type: 'scatter'
    }]
  }
}

// 获取雷达图配置
const getRadarOption = (customIndicator, customData) => {
  const indicator = customIndicator || [
    { name: '销售', max: 6500 },
    { name: '管理', max: 16000 },
    { name: '技术', max: 30000 },
    { name: '客服', max: 38000 },
    { name: '研发', max: 52000 },
    { name: '市场', max: 25000 }
  ]
  
  const data = customData || [
    {
      value: [4200, 3000, 20000, 35000, 50000, 18000],
      name: '预算执行'
    },
    {
      value: [5200, 2000, 23000, 31000, 42000, 21000],
      name: '实际支出'
    }
  ]
  
  return {
    title: {
      text: '雷达图',
      textStyle: {
        fontSize: 14
      }
    },
    tooltip: {},
    legend: {
      data: data.map(d => d.name)
    },
    radar: {
      indicator: indicator
    },
    series: [{
      name: '雷达图数据',
      type: 'radar',
      data: data
    }]
  }
}

// 获取仪表盘配置
const getGaugeOption = (customValue) => {
  const value = customValue !== undefined ? customValue : 75
  
  return {
    title: {
      text: '仪表盘',
      textStyle: {
        fontSize: 14
      }
    },
    tooltip: {},
    series: [{
      name: '业务指标',
      type: 'gauge',
      detail: { formatter: '{value}%' },
      data: [{ value: value, name: '完成率' }]
    }]
  }
}

// 生成随机数据
const generateRandomData = (count, min, max) => {
  const data = []
  for (let i = 0; i < count; i++) {
    data.push(Math.floor(Math.random() * (max - min + 1)) + min)
  }
  return data
}

// 生成散点数据
const generateScatterData = (count) => {
  const data = []
  for (let i = 0; i < count; i++) {
    data.push([
      Math.floor(Math.random() * 100),
      Math.floor(Math.random() * 100)
    ])
  }
  return data
}

// 设置自动刷新
const setupAutoRefresh = (item) => {
  if (refreshTimers[item.id]) {
    clearInterval(refreshTimers[item.id])
    delete refreshTimers[item.id]
  }
  
  if (item.refreshInterval && item.refreshInterval > 0) {
    refreshTimers[item.id] = setInterval(() => {
      const index = dashboardItems.value.findIndex(i => i.id === item.id)
      if (index !== -1) {
        renderChart(item, index)
      }
    }, item.refreshInterval * 1000)
  }
}

// 关闭预览
const closePreview = () => {
  // 如果是在新窗口中
  if (window.opener) {
    window.close()
  } else {
    // 如果是在当前窗口预览，返回上一页
    window.history.back()
  }
}

// 组件卸载时清理
onUnmounted(() => {
  // 清除所有定时器
  Object.values(refreshTimers).forEach(timer => {
    clearInterval(timer)
  })
  
  // 销毁所有图表实例
  Object.values(chartInstances).forEach(instance => {
    instance.dispose()
  })
})
</script>

<style scoped>
.chart-dashboard-preview {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background-color: #f5f7fa;
}

.preview-header {
  padding: 15px 20px;
  background-color: #ffffff;
  border-bottom: 1px solid #e4e7ed;
  display: flex;
  justify-content: space-between;
  align-items: center;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.preview-header h2 {
  margin: 0;
  font-size: 18px;
  font-weight: 500;
  color: #303133;
}

.preview-content {
  flex: 1;
  padding: 20px;
  overflow: auto;
  position: relative;
  background-color: #f5f7fa;
}

.dashboard-item {
  position: absolute;
  background-color: #ffffff;
  border: 1px solid #e4e7ed;
  border-radius: 4px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  transition: all 0.3s ease;
}

.dashboard-item:hover {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.item-header {
  padding: 8px 12px;
  background-color: #f5f7fa;
  border-bottom: 1px solid #e4e7ed;
  font-size: 12px;
}

.item-title {
  font-weight: 500;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  color: #303133;
}

.chart-container {
  width: 100%;
  height: calc(100% - 36px);
}

/* 响应式调整 */
@media (max-width: 768px) {
  .dashboard-item {
    position: relative !important;
    left: 0 !important;
    top: 0 !important;
    margin-bottom: 20px;
    width: 100% !important;
    max-width: 100% !important;
  }
}
</style>