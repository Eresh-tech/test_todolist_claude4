<template>
  <div class="app">
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
      

    </div>

    <!-- 添加任务输入框 -->
    <div class="add-todo" v-if="activeTab === 'todo'">
      <input 
        v-model="newTodo"
        @keyup.enter="addTodo"
        placeholder="添加新任务..."
        class="todo-input"
        @mousedown.stop
      />
      <button @click="addTodo" class="add-btn" @mousedown.stop>+</button>
    </div>

    <!-- 任务列表 -->
    <div class="todo-list">
      <TodoItem
        v-for="(todo, index) in filteredTodos"
        :key="todo.id"
        :todo="todo"
        :class="{
          'dragging': isDraggingTodo && draggedTodoItem?.id === todo.id,
          'drag-over': dragOverIndex === index
        }"
        @toggle="toggleTodo"
        @delete="deleteTodo"
        @set-reminder="setReminder"
        @edit="editTodo"
        @set-due-date="setDueDate"
        @mousedown="handleTodoMouseDown($event, todo)"
      />
    </div>

    <!-- 设置弹窗 -->
    <div v-if="showSettings" class="settings-overlay" @click="showSettings = false" @mousedown.stop>
      <div class="settings-panel" @click.stop @mousedown.stop>
        <h3>设置</h3>
        <div class="setting-item">
          <label>主题颜色</label>
          <input type="color" v-model="themeColor" />
        </div>
        <div class="setting-item">
          <label>自动保存</label>
          <input type="checkbox" v-model="autoSave" />
        </div>
        <div class="setting-item">
          <label>页面拖动</label>
          <input type="checkbox" v-model="enablePageDrag" />
        </div>
        <button @click="showSettings = false" class="close-btn">关闭</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, watch, nextTick } from 'vue'
import TodoItem from './components/TodoItem.vue'

// 重新添加 Tauri API 用于窗口尺寸调整
import { getCurrentWindow } from '@tauri-apps/api/window'

// 响应式数据
const activeTab = ref('todo')
const newTodo = ref('')
const todos = ref([])
const showSettings = ref(false)
const themeColor = ref('#007AFF')
const autoSave = ref(true)

// 移除页面拖动相关状态，使用 CSS 拖动方案

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
const addTodo = async () => {
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
    // ResizeObserver 会自动检测尺寸变化
  }
}

const toggleTodo = async (id) => {
  const todo = todos.value.find(t => t.id === id)
  if (todo) {
    todo.completed = !todo.completed
    saveTodos()
    // ResizeObserver 会自动检测尺寸变化
  }
}

const deleteTodo = async (id) => {
  todos.value = todos.value.filter(t => t.id !== id)
  saveTodos()
  // ResizeObserver 会自动检测尺寸变化
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
onMounted(async () => {
  // 先加载待办事项
  loadTodos()

  // 等待 DOM 渲染完成，确保待办事项都已渲染
  await nextTick()

  // 再等待一段时间确保所有组件都已渲染完成
  setTimeout(() => {
    // 初始化 ResizeObserver
    initResizeObserver()

    // 立即调整窗口尺寸以匹配当前内容
    updateWindowSize()

    console.log('Initial window size adjustment completed')
  }, 200) // 增加延迟时间确保所有内容都已加载
})

onUnmounted(() => {
  cleanupResizeObserver()
})

// 监听器
watch(autoSave, (newVal) => {
  if (newVal) {
    saveTodos()
  }
})

// 监听标签页切换，ResizeObserver 会自动检测尺寸变化
watch(activeTab, async () => {
  await nextTick()
  // ResizeObserver 会自动检测尺寸变化
})

// 添加编辑任务的方法
const editTodo = (id, newText) => {
  const todo = todos.value.find(t => t.id === id)
  if (todo) {
    todo.text = newText
    saveTodos()
  }
}

// 在 <script setup> 中添加新的拖拽实现

// 任务项拖拽相关状态
const draggedTodoItem = ref(null)
const isDraggingTodo = ref(false)
const todoStartPos = ref({ x: 0, y: 0 })
const dragOverIndex = ref(-1)

// 替换原有的拖拽方法
const startTodoDrag = (event, todo) => {
  event.stopPropagation()
  draggedTodoItem.value = todo
  isDraggingTodo.value = true
  todoStartPos.value = { x: event.clientX, y: event.clientY }
  
  // 添加全局鼠标事件监听
  document.addEventListener('mousemove', handleTodoDrag)
  document.addEventListener('mouseup', endTodoDrag)
}

const handleTodoDrag = (event) => {
  if (!isDraggingTodo.value) return
  
  // 计算鼠标位置对应的任务项索引
  const todoElements = document.querySelectorAll('.todo-item')
  let newIndex = -1
  
  for (let i = 0; i < todoElements.length; i++) {
    const rect = todoElements[i].getBoundingClientRect()
    if (event.clientY >= rect.top && event.clientY <= rect.bottom) {
      newIndex = i
      break
    }
  }
  
  dragOverIndex.value = newIndex
}

const endTodoDrag = (event) => {
  if (!isDraggingTodo.value) return
  
  // 移除全局事件监听
  document.removeEventListener('mousemove', handleTodoDrag)
  document.removeEventListener('mouseup', endTodoDrag)
  
  if (dragOverIndex.value >= 0 && draggedTodoItem.value) {
    // 执行拖拽交换逻辑
    const currentTodos = filteredTodos.value
    const draggedIndex = currentTodos.findIndex(t => t.id === draggedTodoItem.value.id)
    const targetIndex = dragOverIndex.value
    
    if (draggedIndex !== targetIndex && draggedIndex >= 0) {
      // 在原数组中找到对应位置并交换
      const allTodos = todos.value
      const draggedTodoInAll = allTodos.findIndex(t => t.id === draggedTodoItem.value.id)
      const targetTodoInAll = allTodos.findIndex(t => t.id === currentTodos[targetIndex].id)
      
      if (draggedTodoInAll >= 0 && targetTodoInAll >= 0) {
        const temp = allTodos[draggedTodoInAll]
        allTodos[draggedTodoInAll] = allTodos[targetTodoInAll]
        allTodos[targetTodoInAll] = temp
        saveTodos()
      }
    }
  }
  
  // 重置状态
  isDraggingTodo.value = false
  draggedTodoItem.value = null
  dragOverIndex.value = -1
}

// 修改 handleTodoMouseDown 方法
const handleTodoMouseDown = (event, todo) => {
  // 检查是否是在任务项的可拖拽区域
  const target = event.target
  if (target.tagName === 'INPUT' || target.tagName === 'BUTTON' || target.closest('button') || target.closest('.action-icons')) {
    return // 如果点击的是输入框或按钮，不启动拖拽
  }

  // 只有在任务项的文本区域才启动任务拖拽
  if (target.closest('.todo-content')) {
    event.stopPropagation()

    // 延迟启动拖拽，避免与点击事件冲突
    setTimeout(() => {
      if (event.buttons === 1) { // 确保鼠标左键仍然按下
        startTodoDrag(event, todo)
      }
    }, 100)
  }
}

// 移除复杂的拖动方法，使用 CSS -webkit-app-region: drag

// ResizeObserver 实例
let resizeObserver = null

// 窗口尺寸自适应功能
const updateWindowSize = async (appElement) => {
  try {
    if (!appElement) {
      appElement = document.querySelector('.app')
    }

    if (appElement) {
      const appWindow = getCurrentWindow()

      // 获取应用元素的实际尺寸
      const rect = appElement.getBoundingClientRect()
      const newWidth = Math.max(300, Math.min(500, rect.width))
      const newHeight = Math.max(400, Math.min(800, rect.height))

      console.log(`App element size: ${rect.width}x${rect.height}`)
      console.log(`Updating window size to: ${newWidth}x${newHeight}`)

      // 获取当前窗口尺寸进行比较
      const currentSize = await appWindow.innerSize()
      console.log(`Current window size: ${currentSize.width}x${currentSize.height}`)

      // 只有当尺寸确实需要改变时才调整
      if (Math.abs(currentSize.width - newWidth) > 5 || Math.abs(currentSize.height - newHeight) > 5) {
        await appWindow.setSize({ width: newWidth, height: newHeight })
        console.log(`Window size updated to: ${newWidth}x${newHeight}`)
      } else {
        console.log('Window size already matches, no update needed')
      }
    }
  } catch (error) {
    console.error('Failed to update window size:', error)
  }
}

// 初始化 ResizeObserver
const initResizeObserver = () => {
  const appElement = document.querySelector('.app')

  if (appElement && window.ResizeObserver) {
    resizeObserver = new ResizeObserver((entries) => {
      for (const entry of entries) {
        // 使用 setTimeout 确保 DOM 更新完成
        setTimeout(() => {
          updateWindowSize(entry.target)
        }, 50)
      }
    })

    resizeObserver.observe(appElement)
    console.log('ResizeObserver initialized')
  } else {
    console.warn('ResizeObserver not supported or app element not found')
  }
}

// 清理 ResizeObserver
const cleanupResizeObserver = () => {
  if (resizeObserver) {
    resizeObserver.disconnect()
    resizeObserver = null
  }
}
</script>

<style scoped>
.app {
  /* 桌面挂件样式 - 矩形版本 */
  width: 350px;
  min-height: 400px;
  background: rgba(40, 44, 52, 0.85);
  border-radius: 0; /* 改为矩形 */
  padding: 0;
  margin: 0;
  position: relative;
  display: flex;
  flex-direction: column;

  /* 毛玻璃效果 */
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
  border: 1px solid rgba(255, 255, 255, 0.1);

  /* 保持拖动功能需要的样式 */
  user-select: none;
}

/* 移除原来的.container样式，因为已经合并到.app中 */

.header {
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 16px 20px;
  background: transparent;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  margin-bottom: 0;

  /* 使标题区域可拖动 */
  -webkit-app-region: drag;
  user-select: none;
}

.tabs {
  display: flex;
  gap: 0;
}

.tab {
  background: transparent;
  border: none;
  color: rgba(255, 255, 255, 0.7);
  font-size: 18px;
  font-weight: 500;
  padding: 8px 16px;
  cursor: pointer;
  transition: all 0.3s ease;
  border-radius: 6px;

  /* 确保按钮可以点击，不被拖动影响 */
  -webkit-app-region: no-drag;
}

.tab.active {
  color: #ffffff;
  font-size: 18px;
  font-weight: 600;
  background: rgba(255, 255, 255, 0.1);
}

.tab:hover {
  color: rgba(255, 255, 255, 0.9);
  background: rgba(255, 255, 255, 0.05);
}

.header-icons {
  display: flex;
  gap: 12px;
}

.icon-btn {
  background: transparent;
  border: none;
  font-size: 16px;
  cursor: pointer;
  padding: 6px;
  border-radius: 6px;
  transition: background 0.3s;
  color: rgba(255, 255, 255, 0.8);
}

.icon-btn:hover {
  background: rgba(255, 255, 255, 0.1);
  color: rgba(255, 255, 255, 1);
}

.add-todo {
  display: flex;
  gap: 10px;
  padding: 20px;
  background: transparent;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.todo-input {
  flex: 1;
  background: rgba(255, 255, 255, 0.1);
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
  border-color: rgba(255, 255, 255, 0.4);
  background: rgba(255, 255, 255, 0.15);
}

.add-btn {
  background: rgba(255, 255, 255, 0.2);
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
  background: rgba(255, 255, 255, 0.3);
}

.todo-list {
  flex: 1;
  overflow-y: auto;
  padding: 0 20px 20px 20px;
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

/* 拖动区域样式 - 使用 CSS 拖动方案 */
.header {
  cursor: grab;
}

.header:active {
  cursor: grabbing;
}

.todo-list .todo-item.dragging {
  opacity: 0.5;
  transform: scale(1.05);
  z-index: 1000;
}

.todo-list .todo-item.drag-over {
  border-top: 2px solid #007AFF;
}

.todo-list .todo-item {
  transition: all 0.2s ease;
}
</style>