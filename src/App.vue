<template>
  <el-config-provider>
    <div class="app-container">
      <el-container>
        <el-header>
          <div class="header-title">
            <h1>Todo Manager</h1>
            <p class="subtitle">管理你的每日待办事项</p>
          </div>
          <div class="header-controls">
            <div class="search-box">
              <el-input
                v-model="searchQuery"
                placeholder="搜索待办事项..."
                prefix-icon="Search"
                clearable
                @input="filterTodos"
              />
            </div>
            <el-select
              v-model="statusFilter"
              placeholder="筛选状态"
              style="width: 120px"
            >
              <el-option label="全部" value="all" />
              <el-option label="未完成" value="active" />
              <el-option label="已完成" value="completed" />
            </el-select>
            <el-button type="primary" size="large" @click="showCreateDialog">
              <el-icon><Plus /></el-icon>新建待办
            </el-button>
          </div>
        </el-header>
        
        <el-main>
          <el-empty v-if="filteredTodos.length === 0" description="暂无待办事项" />
          <div v-else class="todo-list">
            <el-card v-for="todo in filteredTodos" :key="todo.id" class="todo-card" :class="{ 'is-completed': todo.completed }">
              <template #header>
                <div class="card-header">
                  <div class="card-title">
                    <el-checkbox 
                      v-model="todo.completed"
                      @change="toggleTodoStatus(todo)"
                    />
                    <h3 :class="{ 'completed-text': todo.completed }">{{ todo.title }}</h3>
                  </div>
                  <div class="card-actions">
                    <el-button type="primary" link @click="editTodo(todo)">
                      <el-icon><Edit /></el-icon>
                    </el-button>
                    <el-button type="danger" link @click="confirmDelete(todo)">
                      <el-icon><Delete /></el-icon>
                    </el-button>
                  </div>
                </div>
              </template>
              <div class="todo-content">
                <p :class="{ 'completed-text': todo.completed }">{{ todo.description }}</p>
                <div class="todo-details">
                  <el-tag :type="todo.completed ? 'success' : ''">
                    <el-icon><Calendar /></el-icon>
                    {{ formatDateTime(todo.date, todo.time) }}
                  </el-tag>
                  <el-tag v-if="todo.location" :type="todo.completed ? 'success' : 'info'">
                    <el-icon><Location /></el-icon>
                    {{ todo.location }}
                  </el-tag>
                </div>
              </div>
            </el-card>
          </div>
        </el-main>
      </el-container>
    </div>

    <!-- 创建/编辑对话框 -->
    <el-dialog
      v-model="dialogVisible"
      :title="isEditing ? '编辑待办' : '新建待办'"
      width="500px"
    >
      <el-form
        ref="todoForm"
        :model="currentTodo"
        :rules="rules"
        label-width="100px"
      >
        <el-form-item label="标题" prop="title">
          <el-input v-model="currentTodo.title" />
        </el-form-item>
        <el-form-item label="描述" prop="description">
          <el-input
            v-model="currentTodo.description"
            type="textarea"
            :rows="3"
          />
        </el-form-item>
        <el-form-item label="日期" prop="date">
          <el-date-picker
            v-model="currentTodo.date"
            type="date"
            placeholder="选择日期"
            format="YYYY-MM-DD"
          />
        </el-form-item>
        <el-form-item label="时间" prop="time">
          <el-time-picker
            v-model="currentTodo.time"
            placeholder="选择时间"
            format="HH:mm"
          />
        </el-form-item>
        <el-form-item label="地点" prop="location">
          <el-input v-model="currentTodo.location" />
        </el-form-item>
      </el-form>
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="dialogVisible = false">取消</el-button>
          <el-button type="primary" @click="saveTodo">确定</el-button>
        </span>
      </template>
    </el-dialog>
  </el-config-provider>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import dayjs from 'dayjs'
import {
  Calendar,
  Location,
  Edit,
  Delete,
  Plus,
  Search
} from '@element-plus/icons-vue'

const STORAGE_KEY = 'todo-app-items'

const todos = ref([])
const searchQuery = ref('')
const statusFilter = ref('all')
const dialogVisible = ref(false)
const isEditing = ref(false)
const todoForm = ref(null)

// 从本地存储加载数据
const loadTodos = () => {
  const savedTodos = localStorage.getItem(STORAGE_KEY)
  if (savedTodos) {
    try {
      todos.value = JSON.parse(savedTodos)
    } catch (e) {
      console.error('Failed to load todos from localStorage:', e)
      todos.value = []
    }
  }
}

// 保存数据到本地存储
const saveTodos = () => {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(todos.value))
  } catch (e) {
    console.error('Failed to save todos to localStorage:', e)
    ElMessage.error('保存数据失败')
  }
}

// 监听todos的变化，自动保存
watch(todos, () => {
  saveTodos()
}, { deep: true })

// 组件挂载时加载数据
onMounted(() => {
  loadTodos()
})

const currentTodo = ref({
  id: '',
  title: '',
  description: '',
  date: '',
  time: '',
  location: '',
  completed: false
})

const rules = {
  title: [{ required: true, message: '请输入标题', trigger: 'blur' }],
  date: [{ required: true, message: '请选择日期', trigger: 'change' }],
  time: [{ required: true, message: '请选择时间', trigger: 'change' }]
}

const filteredTodos = computed(() => {
  let result = todos.value

  // 根据完成状态筛选
  if (statusFilter.value !== 'all') {
    result = result.filter(todo => 
      statusFilter.value === 'completed' ? todo.completed : !todo.completed
    )
  }

  // 根据搜索关键词筛选
  if (searchQuery.value) {
    const query = searchQuery.value.toLowerCase()
    result = result.filter(todo =>
      todo.title.toLowerCase().includes(query) ||
      todo.description.toLowerCase().includes(query) ||
      todo.location?.toLowerCase().includes(query)
    )
  }

  return result
})

const formatDateTime = (date, time) => {
  if (!date) return ''
  const dateStr = dayjs(date).format('YYYY-MM-DD')
  const timeStr = time ? dayjs(time).format('HH:mm') : ''
  return timeStr ? `${dateStr} ${timeStr}` : dateStr
}

const showCreateDialog = () => {
  isEditing.value = false
  currentTodo.value = {
    id: '',
    title: '',
    description: '',
    date: '',
    time: '',
    location: '',
    completed: false
  }
  dialogVisible.value = true
}

const editTodo = (todo) => {
  isEditing.value = true
  currentTodo.value = { ...todo }
  dialogVisible.value = true
}

const saveTodo = async () => {
  if (!todoForm.value) return
  
  await todoForm.value.validate((valid) => {
    if (valid) {
      if (isEditing.value) {
        const index = todos.value.findIndex(t => t.id === currentTodo.value.id)
        if (index !== -1) {
          todos.value[index] = { ...currentTodo.value }
          ElMessage.success('待办事项已更新')
        }
      } else {
        const newTodo = {
          ...currentTodo.value,
          id: Date.now().toString()
        }
        todos.value.push(newTodo)
        ElMessage.success('待办事项已创建')
      }
      dialogVisible.value = false
    }
  })
}

const confirmDelete = (todo) => {
  ElMessageBox.confirm('确定要删除这个待办事项吗？', '提示', {
    confirmButtonText: '确定',
    cancelButtonText: '取消',
    type: 'warning'
  }).then(() => {
    const index = todos.value.findIndex(t => t.id === todo.id)
    if (index !== -1) {
      todos.value.splice(index, 1)
      ElMessage.success('待办事项已删除')
    }
  }).catch(() => {})
}

// 按日期排序
const sortTodosByDate = () => {
  todos.value.sort((a, b) => {
    const dateA = dayjs(`${a.date} ${a.time || '00:00'}`)
    const dateB = dayjs(`${b.date} ${b.time || '00:00'}`)
    return dateA - dateB
  })
}

// 切换待办事项状态
const toggleTodoStatus = (todo) => {
  const message = todo.completed ? '已完成' : '标记为未完成'
  ElMessage({
    message: `待办事项"${todo.title}"${message}`,
    type: todo.completed ? 'success' : 'info'
  })
  saveTodos()
}
</script>

<style>
* {
  box-sizing: border-box;
}

html, body {
  margin: 0;
  padding: 0;
  overflow-x: hidden;
}

.app-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  min-height: 100vh;
}

.el-header {
  padding: 20px 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 24px;
  height: auto !important;
  margin-bottom: 20px;
}

.header-title {
  text-align: center;
}

.header-title h1 {
  margin: 0;
  font-size: 32px;
  color: #303133;
}

.subtitle {
  margin: 8px 0 0;
  color: #909399;
  font-size: 16px;
}

.header-controls {
  display: flex;
  gap: 16px;
  align-items: center;
  width: 100%;
  max-width: 800px;
}

.search-box {
  flex: 1;
  position: relative;
  min-width: 200px;
}

.todo-list {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 24px;
  padding: 20px 0;
  margin: 0;
  width: 100%;
}

.todo-card {
  transition: transform 0.3s ease, box-shadow 0.3s ease;
  margin: 0;
  transform-origin: center center;
  width: 100%;
  height: 100%;
}

.todo-card .el-card__body {
  height: calc(100% - 60px); /* 减去header的高度 */
  display: flex;
  flex-direction: column;
}

.todo-content {
  display: flex;
  flex-direction: column;
  gap: 12px;
  flex: 1;
}

.todo-details {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  margin-top: auto;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.card-header h3 {
  margin: 0;
  font-size: 18px;
}

.card-actions {
  display: flex;
  gap: 8px;
}

.el-tag {
  display: flex;
  align-items: center;
  gap: 4px;
}

.dialog-footer {
  display: flex;
  justify-content: flex-end;
  gap: 12px;
}

.card-title {
  display: flex;
  align-items: center;
  gap: 12px;
  flex: 1;
  min-width: 0;
}

.card-title h3 {
  margin: 0;
  font-size: 18px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.completed-text {
  text-decoration: line-through;
  color: #909399;
}

.todo-card.is-completed {
  opacity: 0.8;
}

.todo-card.is-completed:hover {
  opacity: 1;
}

@media (max-width: 1200px) {
  .todo-list {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (max-width: 768px) {
  .header-controls {
    flex-direction: column;
    width: 100%;
  }
  
  .search-box, .el-select {
    width: 100% !important;
  }
  
  .el-header {
    gap: 16px;
  }
  
  .todo-list {
    grid-template-columns: 1fr;
  }
  
  .header-title h1 {
    font-size: 28px;
  }
  
  .subtitle {
    font-size: 14px;
  }
}
</style>
