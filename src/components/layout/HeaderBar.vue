<template>
  <header class="header">
    <div class="logo-area">
      <img src="/img/logo.png" alt="Logo" class="logo-img" />
    </div>
    <div class="nav-actions">
      <el-button
        v-for="item in navButtons"
        :key="item.key"
        size="small"
        text
        @click="handleNavClick(item.key)"
      >
        {{ item.label }}
      </el-button>
    </div>
    <div class="user-area">
      <el-button type="primary" round @click="handleLoginClick">登录 / 注册</el-button>
    </div>
  </header>
</template>

<script setup>
import { ElMessage } from 'element-plus';

const navButtons = [
  { key: 'home', label: '首页' },
  { key: 'help', label: '帮助' },
  { key: 'tutorial', label: '引导教程' },
  { key: 'feedback', label: '意见反馈' },
  { key: 'about', label: '关于我们' },
];

const emit = defineEmits(['navigate', 'start-tutorial']);

const handleNavClick = (key) => {
  if (key === 'tutorial') {
    emit('start-tutorial');
  } else {
    emit('navigate', key);
  }
};

const handleLoginClick = () => {
  ElMessage.info('该功能正在开发中，敬请期待！');
};
</script>

<style scoped>
.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 32px;
  height: 64px;
  background: linear-gradient(180deg, #ffffff 0%, #fafbfc 100%);
  box-shadow: 0 2px 8px rgba(31, 35, 51, 0.06), 0 1px 2px rgba(31, 35, 51, 0.04);
  border-bottom: 1px solid rgba(31, 35, 51, 0.06);
  flex-shrink: 0;
  position: relative;
  z-index: 100;
}

.logo-area {
  display: flex;
  align-items: center;
  gap: 16px;
}

.logo-img {
  height: 48px;
  object-fit: contain;
  transition: transform 0.3s ease;
}

.logo-img:hover {
  transform: scale(1.05);
}

.nav-actions {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 100%;
  gap: 2px;
}

.nav-actions :deep(.el-button) {
  font-size: 14px;
  font-weight: 500;
  height: 100%;
  padding: 0 16px;
  border-radius: 6px;
  transition: all 0.2s cubic-bezier(0.4, 0, 0.2, 1);
  color: #64748b;
  position: relative;
}

.nav-actions :deep(.el-button:hover) {
  color: #1f2333;
  background-color: rgba(57, 156, 235, 0.08);
  transform: translateY(-1px);
}

.user-area {
  display: flex;
  align-items: center;
  gap: 16px;
  height: 100%;
}

.user-area :deep(.el-button) {
  height: 40px;
  padding: 0 20px;
  font-weight: 500;
  border-radius: 8px;
  transition: all 0.2s cubic-bezier(0.4, 0, 0.2, 1);
  box-shadow: 0 2px 4px rgba(57, 156, 235, 0.2);
}
</style>

