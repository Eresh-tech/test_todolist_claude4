<template>
  <div class="app">
    <div class="container">
      <!-- 头部标签页 -->
      <div class="header">
        <div class="tabs">
          <button 
            :class="['tab', { active: activeTab === 'todo' }]"
            @click="activeTab = 'todo'"
          >
            Todo
          </button>
          <button 
            :class="['tab', { active: activeTab === 'done' }]"
            @click="activeTab = 'done'"
          >
            Done
          </button>
        </div>
        
        <!-- 右侧图标 -->
        <div class="header-icons">
          <button class="icon-btn" @click="refreshTodos">
            🔄
          </button>
          <button class="icon-btn" @click="showSettings = !showSettings">
            ⚙️
          </button>
        </div>
      </div>

      <!-- 添加任务输入框 -->
      <div class="add-todo" v-if="activeTab === 'todo'">
        <input 
          v-model="newTodo"
          @keyup.enter="addTodo"
          placeholder="添加新任务..."
          class="todo-input"
        />
        <button @click="addTodo" class="add-btn">+</button>
      </div>

      <!-- 任务列表 -->
      <div class="todo-list">
        <TodoItem
          v-for="todo in filteredTodos"
          :key="todo.id"
          :todo="todo"
          draggable="true"
          @dragstart="dragStart($event, todo)"
          @dragover.prevent
          @drop="drop($event, todo)"
          @toggle="toggleTodo"
          @delete="deleteTodo"
          @set-reminder="setReminder"
          @edit="editTodo"
          @set-due-date="setDueDate"
        />
      </div>
    </div>

    <!-- 设置弹窗 -->
    <div v-if="showSettings" class="settings-overlay" @click="showSettings = false">
      <div class="settings-panel" @click.stop>
        <h3>设置</h3>
        <div class="setting-item">
          <label>主题颜色</label>
          <input type="color" v-model="themeColor" />
        </div>
        <div class="setting-item">
          <label>自动保存</label>
          <input type="checkbox" v-model="autoSave" />
        </div>
        <button @click="showSettings = false" class="close-btn">关闭</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import TodoItem from './components/TodoItem.vue'

// 响应式数据
const activeTab = ref('todo')
const newTodo = ref('')
const todos = ref([])
const showSettings = ref(false)
const themeColor = ref('#007AFF')
const autoSave = ref(true)

// 计算属性
const filteredTodos = computed(() => {
  return todos.value.filter(todo => {
    if (activeTab.value === 'todo') {
      return !todo.completed
    } else {
      return todo.completed
    }
  })
})

// 方法
const addTodo = () => {
  if (newTodo.value.trim()) {
    todos.value.push({
      id: Date.now(),
      text: newTodo.value.trim(),
      completed: false,
      createdAt: new Date(),
      reminder: null,
      dueDate: null // 添加截止日期字段
    })
    newTodo.value = ''
    saveTodos()
  }
}

const toggleTodo = (id) => {
  const todo = todos.value.find(t => t.id === id)
  if (todo) {
    todo.completed = !todo.completed
    saveTodos()
  }
}

const deleteTodo = (id) => {
  todos.value = todos.value.filter(t => t.id !== id)
  saveTodos()
}

const setReminder = (id, reminderTime) => {
  const todo = todos.value.find(t => t.id === id)
  if (todo) {
    todo.reminder = reminderTime
    saveTodos()
  }
}

// 添加设置截止日期的方法
const setDueDate = (id, dueDate) => {
  const todo = todos.value.find(t => t.id === id)
  if (todo) {
    todo.dueDate = dueDate
    saveTodos()
  }
}

const refreshTodos = () => {
  loadTodos()
}

const saveTodos = () => {
  if (autoSave.value) {
    localStorage.setItem('todos', JSON.stringify(todos.value))
  }
}

const loadTodos = () => {
  const saved = localStorage.getItem('todos')
  if (saved) {
    todos.value = JSON.parse(saved)
  }
}

// 生命周期
onMounted(() => {
  loadTodos()
})

// 监听器
watch(autoSave, (newVal) => {
  if (newVal) {
    saveTodos()
  }
})

// 添加编辑任务的方法
const editTodo = (id, newText) => {
  const todo = todos.value.find(t => t.id === id)
  if (todo) {
    todo.text = newText
    saveTodos()
  }
}

// 拖曳相关的状态和方法
const draggedTodo = ref(null)

const dragStart = (event, todo) => {
  draggedTodo.value = todo
}

const drop = (event, targetTodo) => {
  event.preventDefault()
  if (!draggedTodo.value || draggedTodo.value.id === targetTodo.id) return

  // 获取源和目标的索引
  const sourceIndex = todos.value.findIndex(t => t.id === draggedTodo.value.id)
  const targetIndex = todos.value.findIndex(t => t.id === targetTodo.id)

  // 交换位置
  const temp = todos.value[sourceIndex]
  todos.value[sourceIndex] = todos.value[targetIndex]
  todos.value[targetIndex] = temp

  draggedTodo.value = null
}
</script>

<style scoped>
.app {
  width: 100vw;
  height: 100vh;
  background: transparent;
  display: flex;
  justify-content: center;
  align-items: flex-start;
  padding: 0px;
}

.container {
  width: 100%;
  max-width: 400px;
  background: rgba(30, 30, 30, 0.65);
  border-radius: 12px;
  padding: 20px;
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.tabs {
  display: flex;
  gap: 0;
}

.tab {
  background: transparent;
  border: none;
  color: rgba(255, 255, 255, 0.6);
  font-size: 20px;
  font-weight: 600;
  padding: 8px 16px;
  cursor: pointer;
  transition: all 0.3s ease;
}

.tab.active {
  color: white;
  font-size: 28px;
  font-weight: 700;
}

.tab:hover {
  color: rgba(255, 255, 255, 0.8);
}

.header-icons {
  display: flex;
  gap: 12px;
}

.icon-btn {
  background: transparent;
  border: none;
  font-size: 18px;
  cursor: pointer;
  padding: 6px;
  border-radius: 6px;
  transition: background 0.3s;
}

.icon-btn:hover {
  background: rgba(255, 255, 255, 0.1);
}

.instructions {
  margin-bottom: 20px;
  font-size: 14px;
  color: rgba(255, 255, 255, 0.7);
}

.instruction-item {
  margin-bottom: 4px;
}

.add-todo {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}

.todo-input {
  flex: 1;
  background: rgba(255, 255, 255, 0.15);
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 8px;
  padding: 12px;
  color: white;
  font-size: 16px;
}

.todo-input::placeholder {
  color: rgba(255, 255, 255, 0.5);
}

.todo-input:focus {
  outline: none;
  border-color: #007AFF;
}

.add-btn {
  background: #007AFF;
  border: none;
  border-radius: 8px;
  color: white;
  font-size: 20px;
  width: 44px;
  height: 44px;
  cursor: pointer;
  transition: background 0.3s;
}

.add-btn:hover {
  background: #0056CC;
}

.todo-list {
  max-height: 400px;
  overflow-y: auto;
}

.settings-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.settings-panel {
  background: rgba(30, 30, 30, 0.85);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
  border-radius: 12px;
  padding: 24px;
  min-width: 300px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.2);
}

.settings-panel h3 {
  color: white;
  margin-bottom: 20px;
  text-align: center;
}

.setting-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
  color: white;
}

.close-btn {
  width: 100%;
  background: #007AFF;
  border: none;
  border-radius: 8px;
  color: white;
  padding: 12px;
  cursor: pointer;
  margin-top: 16px;
}

.todo-list > * {
  cursor: move;
  user-select: none;
}

.todo-list > *:active {
  opacity: 0.8;
}
</style>