<template>
  <section class="panel-card color-panel">
    <!-- 背景配色 -->
    <div class="config-section">
      <div class="section-header">
        <span class="section-title">背景配色</span>
        <span class="section-desc">设置标签云的背景颜色</span>
      </div>
      <div class="section-content">
        <div class="color-item">
          <span class="label">当前背景颜色：</span>
          <el-color-picker
            v-model="localSettings.background"
            @change="handleBackgroundChange"
            @active-change="handleBackgroundChange"
            show-alpha
          />
          <span class="color-preview" :style="{ background: localSettings.background }"></span>
        </div>
      </div>
    </div>

    <!-- 文字配色 -->
    <div class="config-section">
      <div class="section-header">
        <span class="section-title">文字配色</span>
        <span class="section-desc">设置标签文字的颜色方案</span>
      </div>
      <div class="section-content">
        <!-- 当前色带展示 -->
        <div class="ribbon-preview-section">
          <div class="ribbon-header">
            <span class="label">当前色带：</span>
            <el-button 
              size="small" 
              @click="handleColorFlip"
              :icon="Refresh"
            >
              颜色翻转
            </el-button>
          </div>
          <div class="current-ribbon">
            <div
              v-for="(color, index) in currentRibbon"
              :key="`ribbon-${index}`"
              class="ribbon-color-item"
              :style="{ background: color }"
            ></div>
          </div>
        </div>

        <!-- 配色方案选择 -->
        <div class="scheme-selection">
          <div class="scheme-header">
            <span class="label">配色方案：</span>
            <span class="scheme-count">共 {{ availableRibbons.length }} 种方案</span>
          </div>
          <div class="ribbon-gallery">
            <div
              v-for="(scheme, index) in availableRibbons"
              :key="`ribbon-${index}`"
              class="ribbon-scheme-item"
              :class="{ active: currentRibbonIndex === index }"
              @click="handleRibbonSchemeSelect(index)"
            >
              <div class="ribbon-scheme-colors">
                <div
                  v-for="(color, cIndex) in scheme"
                  :key="`scheme-${index}-${cIndex}`"
                  class="scheme-color-block"
                  :style="{ background: color }"
                ></div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </section>
</template>

<script setup>
import { ref, watch, computed, nextTick, onMounted } from 'vue';
import { usePoiStore } from '@/stores/poiStore';
import { Refresh } from '@element-plus/icons-vue';
import { ribbonColorSchemes } from './ribbonColorSchemes';

const poiStore = usePoiStore();

const localSettings = ref({ ...poiStore.colorSettings });
// 只保留4色带
const currentRibbonIndex = ref(1); // 默认第二组配色方案，索引为1

const currentRibbon = computed(() => {
  return localSettings.value.palette || [];
});

const availableRibbons = computed(() => {
  const schemes = ribbonColorSchemes[1] || [];
  return schemes.map(scheme => scheme.map(c => `rgb(${c.join(',')})`));
});

watch(
  () => poiStore.colorSettings,
  (settings) => {
    localSettings.value = { ...settings };
    // 只允许4色带
    nextTick(() => {
      if (availableRibbons.value.length > 0) {
        const paletteStr = JSON.stringify(settings.palette?.map(c => {
          if (c.startsWith('rgb')) return c;
          if (c.startsWith('#')) {
            const hex = c.slice(1);
            const r = parseInt(hex.slice(0, 2), 16);
            const g = parseInt(hex.slice(2, 4), 16);
            const b = parseInt(hex.slice(4, 6), 16);
            return `rgb(${r},${g},${b})`;
          }
          return c;
        }) || []);

        const schemes = availableRibbons.value;
        const index = schemes.findIndex(scheme => {
          const schemeStr = JSON.stringify(scheme);
          return schemeStr === paletteStr;
        });

        if (index !== -1) {
          currentRibbonIndex.value = index;
        } else {
          // 默认第二组方案索引1
          const defaultIndex = 1;
          currentRibbonIndex.value = defaultIndex;
          poiStore.updateColorSettings({
            palette: availableRibbons.value[defaultIndex],
          });
        }
      }
    });
  },
  { immediate: true, deep: true }
);

const handleBackgroundChange = (color) => {
  if (!color) return;
  localSettings.value.background = color;
  poiStore.updateColorSettings({
    background: color,
  });
};

let flipTimer = null;
const handleColorFlip = () => {
  if (flipTimer) clearTimeout(flipTimer);
  flipTimer = setTimeout(() => {
    const reversed = [...currentRibbon.value].reverse();
    localSettings.value.palette = reversed;
    poiStore.updateColorSettings({
      palette: reversed,
      inverted: !localSettings.value.inverted,
    });
  }, 50);
};

const handleRibbonSchemeSelect = (index) => {
  currentRibbonIndex.value = index;
  const selectedScheme = availableRibbons.value[index];
  localSettings.value.palette = selectedScheme;
  poiStore.updateColorSettings({
    palette: selectedScheme,
  });
  // 通知主视图区刷新标签云
  if (poiStore.hasDrawing) {
    // 假定有 handleRenderCloud 方法/动作
    setTimeout(()=>{
      if(window && window.dispatchEvent){
        window.dispatchEvent(new CustomEvent('refreshTagCloud'));
      }
    }, 50);
  }
};

onMounted(() => {
  nextTick(() => {
    if (availableRibbons.value.length > 0) {
      const currentPalette = poiStore.colorSettings.palette || [];
      const paletteStr = JSON.stringify(currentPalette);
      const matched = availableRibbons.value.some((scheme, index) => {
        if (JSON.stringify(scheme) === paletteStr) {
          currentRibbonIndex.value = index;
          return true;
        }
        return false;
      });
      if (!matched) {
        const defaultIndex = 1;
        const defaultScheme = availableRibbons.value[defaultIndex];
        currentRibbonIndex.value = defaultIndex;
        poiStore.updateColorSettings({
          palette: defaultScheme,
        });
      }
    }
  });
});
</script>

<style scoped>
.color-panel {
  min-height: calc(100vh - 160px);
  display: flex;
  flex-direction: column;
  gap: 16px;
  padding: 0px;
  background: #f5f7fa;
}

.config-section {
  background: #fff;
  border-radius: 8px;
  border: 1px solid #e4e7ed;
  overflow: hidden;
}

.section-header {
  padding: 16px 20px;
  background: #fafbfc;
  border-bottom: 1px solid #e4e7ed;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.section-title {
  font-size: 15px;
  font-weight: 600;
  color: #303133;
}

.section-desc {
  font-size: 12px;
  color: #909399;
  margin-left: 12px;
}

.section-content {
  padding: 20px;
}

.color-item {
  display: flex;
  align-items: center;
  gap: 12px;
}

.color-item.spaced {
  justify-content: space-between;
}

.label {
  font-size: 14px;
  color: #606266;
  min-width: 100px;
}

.color-preview {
  width: 48px;
  height: 24px;
  border-radius: 6px;
  border: 1px solid rgba(0, 0, 0, 0.1);
  transition: background 0.2s;
}

/* 当前色带展示 */
.ribbon-preview-section {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 20px;
}

.ribbon-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.current-ribbon {
  display: flex;
  gap: 4px;
  padding: 12px;
  background: #f5f7fa;
  border-radius: 8px;
  border: 1px solid #e4e7ed;
}

.ribbon-color-item {
  flex: 1;
  height: 32px;
  border-radius: 4px;
  border: 1px solid rgba(0, 0, 0, 0.1);
  min-width: 40px;
}

/* 离散设置 */
.discrete-settings {
  display: flex;
  flex-direction: column;
  gap: 16px;
  margin-bottom: 20px;
  padding: 16px;
  background: #fafbfc;
  border-radius: 6px;
}

/* 配色方案选择 */
.scheme-selection {
  margin-top: 8px;
}

.scheme-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 12px;
}

.scheme-count {
  font-size: 12px;
  color: #909399;
}

.ribbon-gallery {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 16px;
  max-height: 400px;
  overflow-y: auto;
  padding: 4px;
}

.ribbon-scheme-item {
  padding: 12px;
  border: 2px solid #e4e7ed;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.15s;
  background: #fff;
}

.ribbon-scheme-item:hover {
  border-color: #409eff;
  box-shadow: 0 2px 8px rgba(64, 158, 255, 0.2);
  transform: translateY(-1px);
}

.ribbon-scheme-item.active {
  border-color: #409eff;
  background: #ecf5ff;
  box-shadow: 0 2px 12px rgba(64, 158, 255, 0.3);
}

.ribbon-scheme-colors {
  display: flex;
  gap: 2px;
  height: 40px;
}

.scheme-color-block {
  flex: 1;
  border-radius: 3px;
  border: 1px solid rgba(0, 0, 0, 0.1);
  min-width: 8px;
}
</style>
