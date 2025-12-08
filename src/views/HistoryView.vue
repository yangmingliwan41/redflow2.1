<template>
  <PageContainer size="xl">
    <PageHeader
      title="历史记录"
      subtitle="查看和管理你的创作历史"
    />

    <div v-if="loading" class="loading-state">
      <div class="spinner"></div>
      <p>加载中...</p>
    </div>

    <div v-else-if="history.length === 0" class="empty-state">
      <svg width="64" height="64" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1">
        <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path>
        <polyline points="14 2 14 8 20 8"></polyline>
      </svg>
      <h3>暂无历史记录</h3>
      <p>开始创作后，你的作品会显示在这里</p>
    </div>

    <div v-else class="history-grid">
      <div
        v-for="item in history"
        :key="item.id"
        class="history-card"
        @click="openDetail(item)"
      >
        <div v-if="item.originalImageUrl || (item.pages && item.pages[0]?.imageUrl)" class="card-image">
          <img :src="item.originalImageUrl || item.pages?.[0]?.imageUrl" alt="Preview" />
          <!-- 类型标签 -->
          <div class="card-type-badge">
            <span v-if="item.topic">📝 文本生成</span>
            <span v-else>🖼️ 图生图</span>
          </div>
        </div>
        <div class="card-content">
          <h4>{{ item.projectName || item.analysis?.name || item.topic || '未命名作品' }}</h4>
          <p class="card-meta">
            {{ new Date(item.createdAt || 0).toLocaleDateString() }}
            <span v-if="item.pages" class="page-count"> · {{ item.pages.length }} 页</span>
          </p>
          <div v-if="item.marketingCopy" class="card-preview">
            {{ item.marketingCopy.substring(0, 100) }}...
          </div>
          <div v-else-if="item.topic" class="card-preview">
            {{ item.topic }}
          </div>
        </div>
      </div>
    </div>

    <!-- 详情弹窗 -->
    <HistoryDetailModal
      :visible="detailModalVisible"
      :item="selectedItem"
      @close="closeDetailModal"
    />
  </PageContainer>
</template>

<script setup lang="ts">
import { ref, onMounted, watch } from 'vue'
import { useRoute } from 'vue-router'
import { getUserHistory, getCurrentUser, registerUser, loginUser } from '../services/storage'
import { GeneratedResult } from '../types'
import HistoryDetailModal from '../components/HistoryDetailModal.vue'
import { PageContainer, PageHeader } from '../components/layout'

const route = useRoute()
const loading = ref(false)
const history = ref<GeneratedResult[]>([])
const detailModalVisible = ref(false)
const selectedItem = ref<GeneratedResult | null>(null)

const loadHistory = () => {
  console.log('=== 开始加载历史记录 ===')
  const user = getCurrentUser()
  
  if (user) {
    console.log('找到用户:', user.id, user.username)
    const userHistory = getUserHistory(user.id)
    console.log('加载历史记录:', {
      userId: user.id,
      count: userHistory.length,
      items: userHistory.map(h => ({
        id: h.id,
        mode: h.mode,
        topic: h.topic,
        projectName: h.projectName,
        createdAt: h.createdAt
      }))
    })
    
    // 检查 localStorage 中的实际数据
    const key = `redflow_history_${user.id}`
    const rawData = localStorage.getItem(key)
    console.log('localStorage 原始数据:', rawData ? `长度: ${rawData.length}` : 'null')
    
    if (rawData) {
      try {
        const parsed = JSON.parse(rawData)
        console.log('解析后的数据:', parsed)
      } catch (e) {
        console.error('解析历史记录数据失败:', e)
      }
    }
    
    history.value = userHistory
    console.log('=== 历史记录加载完成，显示数量:', history.value.length, '===')
  } else {
    console.warn('未找到当前用户，无法加载历史记录')
    // 尝试获取或创建默认用户
    try {
      // 先尝试登录已存在的默认用户
      try {
        const existingUser = loginUser('default@example.com')
        console.log('找到已存在的默认用户:', existingUser.id)
        loginUser(existingUser.email)
        history.value = getUserHistory(existingUser.id)
        console.log('默认用户历史记录数量:', history.value.length)
      } catch (loginError) {
        // 如果登录失败，尝试创建新用户
        console.log('默认用户不存在，尝试创建...')
        const defaultUser = registerUser('default_user', 'default@example.com')
        if (defaultUser) {
          console.log('已创建默认用户:', defaultUser.id)
          loginUser(defaultUser.email)
          history.value = getUserHistory(defaultUser.id)
          console.log('默认用户历史记录数量:', history.value.length)
        }
      }
    } catch (e: any) {
      console.error('处理默认用户失败:', e)
      // 如果邮箱已存在，尝试直接使用该用户
      if (e.message && e.message.includes('Email already exists')) {
        try {
          const usersStr = localStorage.getItem('redflow_users')
          const users = usersStr ? JSON.parse(usersStr) : []
          const defaultUser = users.find((u: any) => u.email === 'default@example.com')
          if (defaultUser) {
            console.log('使用已存在的默认用户:', defaultUser.id)
            loginUser(defaultUser.email)
            history.value = getUserHistory(defaultUser.id)
            console.log('默认用户历史记录数量:', history.value.length)
          }
        } catch (fallbackError) {
          console.error('回退方案也失败:', fallbackError)
        }
      }
    }
  }
}

const openDetail = (item: GeneratedResult) => {
  selectedItem.value = item
  detailModalVisible.value = true
}

const closeDetailModal = () => {
  detailModalVisible.value = false
  selectedItem.value = null
}

const viewDetail = async (item: GeneratedResult) => {
  // 如果是文本生成图文模式，可以选择直接跳转或显示详情
  // 这里我们统一使用详情弹窗，用户可以在弹窗中选择"查看完整结果"
  openDetail(item)
}

onMounted(() => {
  loadHistory()
})

// 监听路由变化，当从其他页面返回时重新加载历史记录
watch(() => route.path, (newPath) => {
  if (newPath === '/history') {
    console.log('=== 路由切换到历史记录页面，重新加载 ===')
    loadHistory()
  }
})
</script>

<style scoped>
.loading-state,
.empty-state {
  text-align: center;
  padding: 80px 20px;
  color: var(--text-sub);
}

.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid var(--border-color);
  border-top-color: var(--primary);
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin: 0 auto 16px;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.history-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 24px;
}

.history-card {
  background: var(--bg-card);
  border-radius: var(--radius-lg);
  overflow: hidden;
  cursor: pointer;
  transition: all 0.2s;
  border: 1px solid var(--border-color);
}

.history-card:hover {
  transform: translateY(-4px);
  box-shadow: var(--shadow-md);
}

.card-image {
  width: 100%;
  height: 200px;
  overflow: hidden;
  background: var(--bg-body);
  position: relative;
}

.card-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.card-type-badge {
  position: absolute;
  top: 8px;
  right: 8px;
  background: rgba(0, 0, 0, 0.6);
  color: white;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 12px;
  font-weight: 500;
}

.card-content {
  padding: 16px;
}

.card-content h4 {
  font-size: 16px;
  font-weight: 600;
  margin-bottom: 8px;
  color: var(--text-main);
}

.card-meta {
  font-size: 12px;
  color: var(--text-secondary);
  margin-bottom: 12px;
}

.page-count {
  color: var(--primary);
  font-weight: 500;
}

.card-preview {
  font-size: 14px;
  color: var(--text-sub);
  line-height: 1.5;
}
</style>

