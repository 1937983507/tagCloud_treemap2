<template>
  <footer class="footer">
    <div class="footer-content">
      <div class="footer-left">
        <div class="footer-brand">
          <img src="/img/logo.png" alt="Logo" class="footer-logo" />
          <div class="brand-info">
            <h3 class="brand-name">标签云 TreeMap</h3>
            <p class="brand-desc">专业的地点可视化平台</p>
          </div>
        </div>
      </div>
      <div class="footer-main">
        <div class="footer-links">
          <a href="javascript:void(0)" @click="handleLinkClick('home')">首页</a>
          <span class="divider">|</span>
          <a href="javascript:void(0)" @click="handleLinkClick('help')">帮助</a>
          <span class="divider">|</span>
          <a href="javascript:void(0)" @click="handleLinkClick('feedback')">意见反馈</a>
        </div>
        <div class="footer-copyright">
          <span>Copyright © 2024 标签云 TreeMap All Rights Reserved.</span>
        </div>
      </div>
    </div>
  </footer>
</template>

<script setup>
const emit = defineEmits(['navigate']);

const handleLinkClick = (key) => {
  emit('navigate', key);
};
</script>

<style scoped>
.footer {
  background: #ffffff;
  color: #1f2333;
  padding: 12px 32px;
  margin-top: auto;
  flex-shrink: 0;
  border-top: 1px solid rgba(31, 35, 51, 0.08);
  box-shadow: 0 -2px 8px rgba(31, 35, 51, 0.04);
}

.footer-content {
  max-width: 1400px;
  margin: 0 auto;
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 40px;
  align-items: center;
}

.footer-left {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: flex-start;
}

.footer-brand {
  display: flex;
  align-items: center;
  gap: 12px;
  height: 100%;
}

.footer-logo {
  height: 40px;
  object-fit: contain;
}

.brand-info {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.brand-name {
  margin: 0;
  font-size: 16px;
  font-weight: 600;
  color: #1f2333;
}

.brand-desc {
  margin: 0;
  font-size: 12px;
  color: #64748b;
}

.footer-main {
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  gap: 12px;
}

.footer-links {
  display: flex;
  align-items: center;
  gap: 12px;
  font-size: 13px;
}

.footer-links a {
  color: #64748b;
  text-decoration: none;
  transition: all 0.2s ease;
  padding: 2px 0;
}

.footer-links a:hover {
  color: #399ceb;
}

.divider {
  color: rgba(31, 35, 51, 0.25);
  font-size: 12px;
}

.footer-copyright {
  display: flex;
  align-items: center;
  justify-content: center;
  flex-wrap: nowrap;
  gap: 4px;
  font-size: 12px;
  color: #64748b;
  text-align: center;
}
</style>

