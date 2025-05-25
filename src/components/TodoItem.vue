<template>
  <div 
    class="todo-item" 
    :class="{ completed: todo.completed }"
    @dragstart="handleDragStart"
    @dragend="handleDragEnd"
  >
    <div class="todo-content" @click="toggleEdit">
      <div v-if="isEditing" class="edit-mode">
        <input 
          type="text" 
          v-model="editedText" 
          @keyup.enter="saveEdit" 
          @blur="saveEdit"
          ref="editInput"
          @click.stop
          @mousedown.stop
        />
      </div>
      <div v-else class="display-mode">
        <span class="todo-text">{{ todo.text }}</span>
        <div class="todo-details">
          <span class="todo-time" v-if="todo.dueDate">
            剩余: {{ getRemainingTime(todo.dueDate) }}
          </span>
          <span class="todo-time" v-else>
            创建于: {{ formatTime(todo.createdAt) }}
          </span>
          <span v-if="todo.reminder" class="reminder-indicator">⏰</span>
        </div>
      </div>
    </div>
    
    <div class="action-icons">
      <button class="icon-btn edit-btn" @click.stop="startEdit" @mousedown.stop title="编辑任务">
        ✏️
      </button>
      <button class="icon-btn delete-btn" @click.stop="$emit('delete', todo.id)" @mousedown.stop title="删除任务">
        🗑️
      </button>
      <button class="icon-btn complete-btn" @click.stop="$emit('toggle', todo.id)" @mousedown.stop title="完成/取消完成任务">
        {{ todo.completed ? '↩️' : '✅' }}
      </button>
      <div class="calendar-wrapper">
        <button class="icon-btn calendar-btn" @click.stop="openDatePicker" @mousedown.stop title="设置截止日期">
          📅
        </button>
        
        <!-- 隐藏的日期选择器，定位在按钮附近 -->
        <input 
          type="date" 
          ref="datePicker"
          @change="updateDueDate"
          @click.stop
          @mousedown.stop
          :value="todo.dueDate ? formatDateForInput(todo.dueDate) : ''"
          class="hidden-date-picker"
        />
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue';

const props = defineProps({
  todo: {
    type: Object,
    required: true
  }
});

const emit = defineEmits(['toggle', 'delete', 'edit', 'set-due-date']);

const isEditing = ref(false);
const editedText = ref('');
const editInput = ref(null);
const datePicker = ref(null);

// 处理拖动开始事件，阻止冒泡
const handleDragStart = (event) => {
  event.stopPropagation()
  // 这里可以添加任务项拖动的逻辑
}

// 处理拖动结束事件
const handleDragEnd = (event) => {
  event.stopPropagation()
}

const toggleEdit = () => {
  // 点击整个项目不再触发编辑模式
  // 编辑模式现在只通过点击编辑按钮触发
};

const startEdit = () => {
  if (!isEditing.value) {
    isEditing.value = true;
    editedText.value = props.todo.text;
    nextTick(() => {
      editInput.value.focus();
    });
  }
};

const saveEdit = () => {
  if (isEditing.value) {
    isEditing.value = false;
    if (editedText.value.trim() !== '') {
      emit('edit', props.todo.id, editedText.value);
    }
  }
};

const formatTime = (timestamp) => {
  const date = new Date(timestamp);
  return date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
};

const getRemainingTime = (dueDate) => {
  const now = new Date();
  // 设置now的时间为00:00:00，只比较日期部分
  now.setHours(0, 0, 0, 0);
  
  const due = new Date(dueDate);
  // 设置due的时间为00:00:00，只比较日期部分
  due.setHours(0, 0, 0, 0);
  
  const diffMs = due - now;
  
  if (diffMs < 0) {
    return '已过期';
  }
  
  // 计算天数差异（向上取整到天）
  const diffDays = Math.ceil(diffMs / (1000 * 60 * 60 * 24));
  
  if (diffDays === 0) {
    return '今天到期';
  } else {
    return `${diffDays}天`;
  }
};

const formatDateForInput = (dateString) => {
  const date = new Date(dateString);
  // 只返回YYYY-MM-DD格式的日期部分
  return date.toISOString().split('T')[0];
};

const openDatePicker = () => {
  nextTick(() => {
    if (datePicker.value) {
      datePicker.value.showPicker();
    }
  });
};

const updateDueDate = (event) => {
  const newDueDate = event.target.value;
  if (newDueDate) {
    // 创建日期对象，设置为当天的00:00:00
    const date = new Date(newDueDate);
    date.setHours(0, 0, 0, 0);
    emit('set-due-date', props.todo.id, date.toISOString());
  } else {
    emit('set-due-date', props.todo.id, null);
  }
};
</script>

<style scoped>
.todo-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 16px;
  background: transparent; /* 将背景设置为透明 */
  border-radius: 8px;
  margin-bottom: 10px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  position: relative;
  overflow: visible;
  transition: all 0.3s ease;
}

.todo-item:hover {
  background: rgba(255, 255, 255, 0.1); /* 鼠标悬停时可以稍微改变背景以提供视觉反馈 */
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.15);
}

.todo-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  cursor: pointer;
}

.todo-item.completed .todo-text {
  text-decoration: line-through;
  color: #888;
}

.todo-text {
  font-size: 16px;
  margin-bottom: 4px;
  word-break: break-word;
}

.todo-details {
  display: flex;
  align-items: center;
  gap: 8px;
}

.todo-time {
  font-size: 12px;
  color: white; /* 将字体颜色改为白色 */
}

.reminder-indicator {
  font-size: 12px;
}

.edit-mode input {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 16px;
}

.action-icons {
  display: flex;
  align-items: center;
  gap: 8px;
  opacity: 0;
  transition: opacity 0.2s ease;
}

.todo-item:hover .action-icons {
  opacity: 1;
}

.icon-btn {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  border: none;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.2s ease;
  background: #f5f5f5;
}

.icon-btn:hover {
  transform: scale(1.1);
}

.edit-btn {
  background: #64b5f6;
  color: white;
}

.delete-btn {
  background: #e57373;
  color: white;
}

.complete-btn {
  background: #81c784;
  color: white;
}

.calendar-btn {
  background: #ffb74d;
  color: white;
}

/* Calendar wrapper and hidden date picker positioning */
.calendar-wrapper {
  position: relative;
}

.hidden-date-picker {
  position: absolute;
  top: 0;
  left: 0;
  width: 1px;
  height: 1px;
  opacity: 0;
  pointer-events: none;
}
</style>