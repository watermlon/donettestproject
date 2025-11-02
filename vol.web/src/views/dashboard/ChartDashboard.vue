<template>
  <div class="chart-dashboard-container">
    <!-- 左侧图表组件列表 -->
    <div class="left-panel">
      <div class="panel-header">
        <h3>图表组件</h3>
      </div>
      <div class="chart-components-list">
        <div 
          v-for="(component, index) in chartComponents" 
          :key="index"
          class="chart-component-item"
          draggable="true"
          @dragstart="handleDragStart($event, component)"
        >
          <i :class="component.icon"></i>
          <span>{{ component.name }}</span>
        </div>
      </div>
    </div>

    <!-- 中央设计区域 -->
    <div 
      class="design-area"
      @dragover.prevent
      @drop="handleDrop($event)"
    >
      <div class="design-header">
        <h3>工作台设计区</h3>
        <div class="design-actions">
          <el-button type="primary" size="small" @click="saveDashboard">保存</el-button>
          <el-button size="small" @click="loadDashboard">加载</el-button>
          <el-button size="small" @click="previewDashboard">预览</el-button>
          <el-button size="small" @click="clearDashboard">清空</el-button>
        </div>
      </div>
      <div class="dashboard-grid">
        <div 
          v-for="(item, index) in dashboardItems" 
          :key="item.id || index"
          class="dashboard-item"
          :style="{
              width: item.width || '300px',
              height: item.height || '260px',
              left: item.left || `${(index % 3) * 320}px`,
              top: item.top || `${Math.floor(index / 3) * 280}px`
            }"
          @click="selectItem(item, index)"
          @contextmenu.prevent="showContextMenu($event, item, index)"
        >
          <div class="item-header">
            <span class="item-title">{{ item.title || item.name }}</span>
            <div class="item-actions">
              <i class="el-icon-close" @click.stop="removeItem(index)"></i>
            </div>
          </div>
          <div 
            class="chart-container" 
            :id="`chart-${item.id || index}`"
            ref="chartRefs"
          ></div>
        </div>
      </div>
    </div>

    <!-- 右侧属性配置面板 -->
    <div v-if="selectedItem" class="right-panel">
      <div class="panel-header">
        <h3>属性配置</h3>
      </div>
      <div class="properties-form">
        <el-form label-position="top">
          <el-form-item label="标题">
            <el-input v-model="selectedItem.title" placeholder="请输入图表标题"></el-input>
          </el-form-item>
          <el-form-item label="宽度">
            <el-input-number v-model="selectedItem.width" :min="100" :max="1000" :step="10"></el-input-number>
          </el-form-item>
          <el-form-item label="高度">
            <el-input-number v-model="selectedItem.height" :min="100" :max="800" :step="10"></el-input-number>
          </el-form-item>
          <el-form-item label="数据类型">
            <el-select v-model="selectedItem.dataType" placeholder="请选择数据类型">
              <el-option label="模拟数据" value="mock"></el-option>
              <el-option label="API数据" value="api"></el-option>
            </el-select>
          </el-form-item>
          <el-form-item v-if="selectedItem.dataType === 'api'" label="API地址">
            <el-input v-model="selectedItem.apiUrl" placeholder="请输入API地址"></el-input>
          </el-form-item>
          <el-form-item v-if="selectedItem.type === 'line' || selectedItem.type === 'bar'" label="图表类型">
            <el-select v-model="selectedItem.subType" placeholder="请选择图表类型">
              <el-option label="折线图" value="line"></el-option>
              <el-option label="柱状图" value="bar"></el-option>
              <el-option label="混合图" value="mix"></el-option>
            </el-select>
          </el-form-item>
          <el-form-item v-if="selectedItem.type === 'pie'" label="饼图类型">
            <el-select v-model="selectedItem.subType" placeholder="请选择饼图类型">
              <el-option label="普通饼图" value="pie"></el-option>
              <el-option label="环形图" value="doughnut"></el-option>
            </el-select>
          </el-form-item>
          <el-form-item label="刷新间隔(秒)">
            <el-input-number v-model="selectedItem.refreshInterval" :min="0" :max="3600"></el-input-number>
          </el-form-item>
          <el-form-item>
            <el-button type="primary" @click="updateChart">更新图表</el-button>
          </el-form-item>
        </el-form>
      </div>
    </div>

    <!-- 右键菜单 -->
    <div 
      v-if="contextMenuVisible" 
      class="context-menu"
      :style="{ left: contextMenuX + 'px', top: contextMenuY + 'px' }"
    >
      <div class="menu-item" @click="removeItem(contextMenuIndex)">删除</div>
      <div class="menu-item" @click="duplicateItem(contextMenuIndex)">复制</div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, nextTick, watch, onUnmounted } from 'vue'
import * as echarts from 'echarts'
import { ElMessage } from 'element-plus'
import http from '@/api/http.js'

// 图表组件列表
const chartComponents = [
  { name: '折线图', type: 'line', icon: 'el-icon-pie-chart' },
  { name: '柱状图', type: 'bar', icon: 'el-icon-data-line' },
  { name: '饼图', type: 'pie', icon: 'el-icon-data-analysis' },
  { name: '散点图', type: 'scatter', icon: 'el-icon-data-line' },
  { name: '雷达图', type: 'radar', icon: 'el-icon-data-line' },
  { name: '仪表盘', type: 'gauge', icon: 'el-icon-data-line' }
]

// 工作台项目列表
const dashboardItems = ref([])
// 选中的项目
const selectedItem = ref(null)
// 图表实例引用
const chartInstances = reactive({})
const chartRefs = ref([])
// 右键菜单
const contextMenuVisible = ref(false)
const contextMenuX = ref(0)
const contextMenuY = ref(0)
const contextMenuIndex = ref(-1)
// 自动刷新定时器
const refreshTimers = reactive({})
// 仪表盘ID，用于保存和加载
const dashboardId = ref('')
// 预览模式
const previewMode = ref(false)

// 拖拽开始
const handleDragStart = (event, component) => {
  event.dataTransfer.setData('chartComponent', JSON.stringify(component))
}

// 拖拽放置
const handleDrop = (event) => {
  const componentData = event.dataTransfer.getData('chartComponent')
  if (componentData) {
    const component = JSON.parse(componentData)
    const newItem = {
      id: Date.now(),
      name: component.name,
      type: component.type,
      title: `${component.name}-${dashboardItems.value.length + 1}`,
      width: '300px',
      height: '250px',
      left: `${event.offsetX - 150}px`,
      top: `${event.offsetY - 125}px`,
      dataType: 'mock',
      apiUrl: '',
      refreshInterval: 0,
      subType: component.type
    }
    dashboardItems.value.push(newItem)
    
    nextTick(() => {
      renderChart(newItem, dashboardItems.value.length - 1)
    })
  }
}

// 选择项目
const selectItem = (item, index) => {
  selectedItem.value = item
}

// 显示右键菜单
const showContextMenu = (event, item, index) => {
  event.preventDefault()
  contextMenuX.value = event.clientX
  contextMenuY.value = event.clientY
  contextMenuIndex.value = index
  contextMenuVisible.value = true
  
  document.addEventListener('click', hideContextMenu)
}

// 隐藏右键菜单
const hideContextMenu = () => {
  contextMenuVisible.value = false
  document.removeEventListener('click', hideContextMenu)
}

// 删除项目
const removeItem = (index) => {
  if (chartInstances[`chart-${dashboardItems.value[index].id}`]) {
    chartInstances[`chart-${dashboardItems.value[index].id}`].dispose()
    delete chartInstances[`chart-${dashboardItems.value[index].id}`]
  }
  dashboardItems.value.splice(index, 1)
  if (selectedItem.value && dashboardItems.value[index] === selectedItem.value) {
    selectedItem.value = null
  }
  contextMenuVisible.value = false
}

// 复制项目
const duplicateItem = (index) => {
  const original = dashboardItems.value[index]
  const newItem = {
    ...JSON.parse(JSON.stringify(original)),
    id: Date.now(),
    title: `${original.title}-副本`,
    left: `${parseInt(original.left) + 20}px`,
    top: `${parseInt(original.top) + 20}px`
  }
  dashboardItems.value.push(newItem)
  
  nextTick(() => {
    renderChart(newItem, dashboardItems.value.length - 1)
  })
  
  contextMenuVisible.value = false
}

// 获取图表数据
const fetchChartData = async (item) => {
  if (item.dataType === 'api' && item.apiUrl) {
    try {
      // 从API获取数据
      const response = await http.post(item.apiUrl, {})
      if (response.status || response.success) {
        return response.data || response.result
      }
      ElMessage.warning('API数据获取失败：' + (response.message || '未知错误'))
    } catch (error) {
      console.error('API请求失败:', error)
      ElMessage.error('API请求失败：' + error.message)
    }
  }
  
  // 返回模拟数据
  return null
}

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
  // 根据API返回的数据结构调整图表配置
  // 这里假设API返回的数据格式与我们需要的格式匹配
  // 实际应用中需要根据后端返回的数据结构进行适配
  if (apiData && apiData.chartOption) {
    // 如果API直接返回完整的图表配置
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
  
  // 根据数据类型构建配置
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
    // 使用自定义数据系列
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
      data: dateArr.slice(-7), // 只显示最近7天
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

// 更新图表
const updateChart = () => {
  if (selectedItem.value) {
    const index = dashboardItems.value.findIndex(item => item.id === selectedItem.value.id)
    if (index !== -1) {
      renderChart(selectedItem.value, index)
      ElMessage.success('图表已更新')
    }
  }
}

// 设置自动刷新
const setupAutoRefresh = (item) => {
  // 清除已存在的定时器
  if (refreshTimers[item.id]) {
    clearInterval(refreshTimers[item.id])
    delete refreshTimers[item.id]
  }
  
  // 设置新的定时器
  if (item.refreshInterval && item.refreshInterval > 0) {
    refreshTimers[item.id] = setInterval(() => {
      const index = dashboardItems.value.findIndex(i => i.id === item.id)
      if (index !== -1) {
        renderChart(item, index)
      }
    }, item.refreshInterval * 1000)
  }
}

// 保存工作台
const saveDashboard = async () => {
    try {
      // 弹出对话框让用户输入仪表盘名称
      ElMessageBox.prompt('请输入仪表盘名称', '保存仪表盘', {
        confirmButtonText: '确定',
        cancelButtonText: '取消'
      }).then(async ({ value }) => {
        // 先获取可能存在的保存配置，用于保留createTime
        const savedConfigStr = localStorage.getItem('chartDashboardConfig');
        let savedConfig = null;
        try {
          savedConfig = savedConfigStr ? JSON.parse(savedConfigStr) : null;
        } catch (e) {
          console.error('解析保存的配置失败:', e);
        }
        
        // 创建新的仪表盘配置
        const dashboardConfig = {
          id: dashboardId.value || Date.now().toString(),
          name: value || `仪表盘_${new Date().toLocaleString()}`,
          items: dashboardItems.value,
          timestamp: Date.now(),
          createTime: dashboardId.value && savedConfig && savedConfig.createTime ? 
            savedConfig.createTime : new Date().toISOString(),
          updateTime: new Date().toISOString()
        }
        
        // 保存到后端
        const response = await http.post('/api/chartDashboard/save', dashboardConfig, '正在保存工作台...')
        
        if (response.status || response.success) {
          // 更新dashboardId
          dashboardId.value = response.data?.id || dashboardConfig.id
          
          // 同时保存到localStorage作为备份
          localStorage.setItem('chartDashboardConfig', JSON.stringify(dashboardConfig))
          
          ElMessage.success('工作台已成功保存到服务器')
        } else {
          // 保存失败，尝试仅保存到localStorage
          localStorage.setItem('chartDashboardConfig', JSON.stringify(dashboardConfig))
          ElMessage.warning('服务器保存失败，已保存到本地：' + (response.message || '未知错误'))
        }
      }).catch(() => {
        // 用户取消保存
      })
    } catch (error) {
      console.error('保存工作台失败:', error)
      ElMessage.error('保存失败，请稍后重试')
    }
  }

// 预览工作台
const previewDashboard = () => {
  // 切换到预览模式
  previewMode.value = true
  
  // 在实际应用中，可以打开一个新窗口或全屏预览
  const previewContent = {
    items: dashboardItems.value,
    timestamp: Date.now()
  }
  
  // 保存预览配置到sessionStorage
  sessionStorage.setItem('chartDashboardPreview', JSON.stringify(previewContent))
  
  // 打开预览窗口
  const previewUrl = `${window.location.origin}${window.location.pathname}#/chartDashboard/preview`
  window.open(previewUrl, '_blank')
  
  // 3秒后恢复编辑模式
  setTimeout(() => {
    previewMode.value = false
  }, 3000)
}

// 清空工作台
const clearDashboard = () => {
  // 清除所有定时器
  Object.values(refreshTimers).forEach(timer => {
    clearInterval(timer)
  })
  Object.keys(refreshTimers).forEach(key => {
    delete refreshTimers[key]
  })
  
  // 销毁所有图表实例
  Object.values(chartInstances).forEach(instance => {
    instance.dispose()
  })
  Object.keys(chartInstances).forEach(key => {
    delete chartInstances[key]
  })
  
  // 清空工作台项目
  dashboardItems.value = []
  selectedItem.value = null
  dashboardId.value = ''
  
  // 清除localStorage
  localStorage.removeItem('chartDashboardConfig')
  
  ElMessage.success('工作台已清空')
}

// 加载仪表盘
const loadDashboard = async () => {
  try {
    // 从后端加载仪表盘配置
    const response = await http.post('/api/chartDashboard/getList', {}, '正在加载仪表盘列表...')
    
    if (response.status && response.data && response.data.length > 0) {
      // 这里可以实现一个选择对话框，让用户选择要加载的仪表盘
      // 为简化实现，我们默认加载第一个仪表盘
      const dashboard = response.data[0]
      
      // 加载选中的仪表盘详细配置
      const detailResponse = await http.post('/api/chartDashboard/getById', { id: dashboard.id }, '正在加载仪表盘...')
      
      if (detailResponse.status && detailResponse.data) {
        // 先清空当前工作台
        clearDashboard()
        
        // 设置新的仪表盘配置
        dashboardId.value = detailResponse.data.id
        dashboardItems.value = detailResponse.data.items || []
        
        // 延迟渲染图表
        setTimeout(() => {
          dashboardItems.value.forEach((item, index) => {
            renderChart(item, index)
          })
        }, 100)
        
        ElMessage.success('仪表盘加载成功')
      } else {
        // 如果无法从后端加载，尝试从localStorage加载
        loadFromLocalStorage()
      }
    } else {
      // 如果无法从后端加载，尝试从localStorage加载
      loadFromLocalStorage()
    }
  } catch (error) {
    console.error('加载仪表盘失败:', error)
    // 异常情况下尝试从localStorage加载
    loadFromLocalStorage()
  }
}

// 从localStorage加载
const loadFromLocalStorage = () => {
  const savedConfig = localStorage.getItem('chartDashboardConfig')
  if (savedConfig) {
    try {
      const config = JSON.parse(savedConfig)
      // 先清空当前工作台
      clearDashboard()
      
      // 设置仪表盘配置
      dashboardId.value = config.id
      dashboardItems.value = config.items || []
      
      // 延迟渲染图表
      setTimeout(() => {
        dashboardItems.value.forEach((item, index) => {
          renderChart(item, index)
        })
      }, 100)
      
      ElMessage.success('已从本地加载仪表盘配置')
    } catch (error) {
      console.error('加载本地配置失败:', error)
      ElMessage.warning('加载失败，本地配置无效')
    }
  } else {
    ElMessage.info('没有找到可加载的仪表盘配置')
  }
}

// 监听工作台项目变化，重新渲染图表
watch(() => dashboardItems.value.length, () => {
  // 延迟执行以确保DOM已更新
  setTimeout(() => {
    dashboardItems.value.forEach((item, index) => {
      renderChart(item, index)
    })
  }, 100)
})

// 监听选中项目的刷新间隔变化
watch(() => selectedItem.value?.refreshInterval, () => {
  if (selectedItem.value) {
    setupAutoRefresh(selectedItem.value)
  }
})

// 组件卸载时清理
onUnmounted(() => {
  // 清除所有定时器
  Object.values(refreshTimers).forEach(timer => {
    clearInterval(timer)
  })
  Object.keys(refreshTimers).forEach(key => {
    delete refreshTimers[key]
  })
  
  // 销毁所有图表实例
  Object.values(chartInstances).forEach(instance => {
    instance.dispose()
  })
  Object.keys(chartInstances).forEach(key => {
    delete chartInstances[key]
  })
  
  console.log('图表工作台组件已卸载，所有资源已清理')
})

// 监听选中项目的尺寸变化
watch(() => [selectedItem.value?.width, selectedItem.value?.height], () => {
  if (selectedItem.value) {
    const chartId = `chart-${selectedItem.value.id}`
    if (chartInstances[chartId]) {
      setTimeout(() => {
        chartInstances[chartId].resize()
      }, 100)
    }
  }
})

// 组件挂载时尝试加载保存的配置
onMounted(() => {
  // 组件挂载时自动尝试加载仪表盘配置
  loadDashboard()
  
  // 点击其他区域隐藏属性面板
  document.addEventListener('click', (event) => {
    const dashboardContainer = event.target.closest('.chart-dashboard-container')
    const itemElement = event.target.closest('.dashboard-item')
    const rightPanel = event.target.closest('.right-panel')
    const contextMenu = event.target.closest('.context-menu')
    
    if (dashboardContainer && !itemElement && !rightPanel && !contextMenu) {
      selectedItem.value = null
    }
  })
})
</script>

<style scoped>
.chart-dashboard-container {
  display: flex;
  height: 100vh;
  background-color: #f5f7fa;
  position: relative;
}

/* 左侧面板 */
.left-panel {
  width: 200px;
  background-color: #ffffff;
  border-right: 1px solid #e4e7ed;
  display: flex;
  flex-direction: column;
}

.panel-header {
  padding: 15px;
  border-bottom: 1px solid #e4e7ed;
}

.panel-header h3 {
  margin: 0;
  font-size: 16px;
  font-weight: 500;
}

.chart-components-list {
  padding: 10px;
  overflow-y: auto;
  flex: 1;
}

.chart-component-item {
  display: flex;
  align-items: center;
  padding: 10px;
  margin-bottom: 8px;
  background-color: #f5f7fa;
  border: 1px solid #dcdfe6;
  border-radius: 4px;
  cursor: move;
  transition: all 0.3s;
}

.chart-component-item:hover {
  background-color: #ecf5ff;
  border-color: #409eff;
}

.chart-component-item i {
  margin-right: 8px;
  color: #409eff;
}

/* 中央设计区域 */
.design-area {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.design-header {
  padding: 15px;
  background-color: #ffffff;
  border-bottom: 1px solid #e4e7ed;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.design-header h3 {
  margin: 0;
  font-size: 16px;
  font-weight: 500;
}

.design-actions {
  display: flex;
  gap: 8px;
}

.dashboard-grid {
  flex: 1;
  padding: 20px;
  overflow: auto;
  position: relative;
}

.dashboard-item {
  position: absolute;
  background-color: #ffffff;
  border: 1px solid #e4e7ed;
  border-radius: 4px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  transition: all 0.3s;
  cursor: pointer;
}

.dashboard-item:hover {
  border-color: #409eff;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.dashboard-item.selected {
  border-color: #409eff;
  box-shadow: 0 0 0 2px rgba(64, 158, 255, 0.2);
}

.item-header {
  padding: 8px 12px;
  background-color: #f5f7fa;
  border-bottom: 1px solid #e4e7ed;
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 12px;
}

.item-title {
  font-weight: 500;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  flex: 1;
}

.item-actions i {
  cursor: pointer;
  color: #909399;
  transition: color 0.3s;
}

.item-actions i:hover {
  color: #f56c6c;
}

.chart-container {
  width: 100%;
  height: calc(100% - 36px);
}

/* 右侧面板 */
.right-panel {
  width: 300px;
  background-color: #ffffff;
  border-left: 1px solid #e4e7ed;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.properties-form {
  padding: 15px;
  overflow-y: auto;
  flex: 1;
}

/* 右键菜单 */
.context-menu {
  position: fixed;
  background-color: #ffffff;
  border: 1px solid #e4e7ed;
  border-radius: 4px;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
  z-index: 1000;
  min-width: 120px;
}

.menu-item {
  padding: 8px 16px;
  cursor: pointer;
  transition: background-color 0.3s;
  font-size: 14px;
}

.menu-item:hover {
  background-color: #f5f7fa;
}

.menu-item:not(:last-child) {
  border-bottom: 1px solid #ebeef5;
}
</style>