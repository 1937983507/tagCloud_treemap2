<template>
  <section class="panel-card typeface-panel">
    <!-- 字号设置 -->
    <div class="config-section">
      <div class="section-header">
        <span class="section-title">字号区间</span>
        <span class="section-desc">设置标签字号的最大/最小值</span>
      </div>
      <div class="section-content">
        <div class="fontsize-row">
          <span class="fontsize-label">最小字号：</span>
          <el-input-number v-model="localSettings.minFontSize" :min="2" :max="120" :step="2" @change="() => poiStore.updateFontLevel({ minFontSize: localSettings.minFontSize })" />
          <span class="fontsize-unit">px</span>
        </div>
        <div class="fontsize-row">
          <span class="fontsize-label">最大字号：</span>
          <el-input-number v-model="localSettings.maxFontSize" :min="2" :max="120" :step="2" @change="() => poiStore.updateFontLevel({ maxFontSize: localSettings.maxFontSize })" />
          <span class="fontsize-unit">px</span>
        </div>
      </div>
    </div>

    <!-- 字重选择 -->
    <div class="config-section">
      <div class="section-header">
        <span class="section-title">字重选择</span>
        <span class="section-desc">设置标签的字体粗细</span>
      </div>
      <div class="section-content">
        <el-select
          v-model="localSettings.fontWeight"
          style="width: 200px"
          @change="handleWeightChange"
        >
          <el-option label="Thin 100" value="100" />
          <el-option label="Light 300" value="300" />
          <el-option label="Regular 400" value="400" />
          <el-option label="Medium 500" value="500" />
          <el-option label="Semibold 600" value="600" />
          <el-option label="Bold 700" value="700" />
          <el-option label="Extra Bold 800" value="800" />
          <el-option label="Black 900" value="900" />
        </el-select>
      </div>
    </div>

    <!-- 字体库 -->
    <div class="config-section">
      <div class="section-header">
        <span class="section-title">字体库</span>
        <span class="section-desc">选择标签使用的字体</span>
      </div>
      <div class="section-content">
        <el-tabs v-model="activeFontTab" stretch class="font-tabs">
          <el-tab-pane
            v-for="group in fontGroups"
            :key="group.key"
            :label="group.label"
            :name="group.key"
          >
            <div class="font-gallery">
              <button
                v-for="font in group.fonts"
                :key="font"
                class="font-chip"
                :style="{ fontFamily: font }"
                :class="{ active: poiStore.fontSettings.fontFamily === font }"
                @click="handleFamilyChange(font)"
              >
                <span class="font-name">{{ font }}</span>
                <span 
                  v-if="poiStore.fontSettings.fontFamily === font" 
                  class="font-active-badge"
                >
                  使用中
                </span>
              </button>
            </div>
          </el-tab-pane>
        </el-tabs>
      </div>
    </div>
  </section>
</template>

<script setup>
import { reactive, ref } from 'vue';
import { usePoiStore } from '@/stores/poiStore';

const poiStore = usePoiStore();
const activeFontTab = ref('cn');

const localSettings = reactive({
  minFontSize: poiStore.fontSettings.minFontSize,
  maxFontSize: poiStore.fontSettings.maxFontSize,
  fontWeight: poiStore.fontSettings.fontWeight,
});

const fontGroups = [
  {
    key: 'cn',
    label: '中文',
    fonts: [
      '等线', '等线 Light', '方正舒体', '方正姚体', '仿宋', '黑体',
      '华文彩云', '华文仿宋', '华文琥珀', '华文楷体', '华文隶书', '华文宋体', 
      '华文细黑', '华文新魏', '华文行楷', '华文中宋', '楷体', '隶书', 
      '宋体', '微软雅黑', '微软雅黑 Light', '新宋体', '幼圆', 'Source Han Sans',
      '思源黑体', 'LXGW WenKai Screen', 'ZCOOL KuaiLe'
    ],
  },
  {
    key: 'en',
    label: '英文',
    fonts: [
      'Inter', 'Montserrat', 'Roboto', 'Lato', 'Arial', 'Arial Black',
      'Times New Roman', 'Georgia', 'Verdana', 'Courier New', 'Comic Sans MS',
      'Impact', 'Trebuchet MS', 'Palatino', 'Garamond', 'Bookman',
      'Helvetica', 'Tahoma', 'Lucida Console', 'Century Gothic', 'Franklin Gothic',
      'Baskerville', 'Bodoni', 'Futura', 'Gill Sans', 'Optima'
    ],
  },
  {
    key: 'other',
    label: '其他',
    fonts: [
      'Cormorant Garamond', 'Playfair Display', 'Lora', 'Merriweather',
      'Open Sans', 'Raleway', 'Poppins', 'Nunito', 'Ubuntu', 'Oswald'
    ],
  },
];

function handleWeightChange() {
  poiStore.updateFontLevel({ fontWeight: localSettings.fontWeight });
  // 只有在已绘制路线的情况下才触发重绘（TagCloudCanvas.vue 中的 watch 会自动处理）
}
function handleFamilyChange(font) {
  poiStore.updateFontLevel({ fontFamily: font });
  // 只有在已绘制路线的情况下才触发重绘（TagCloudCanvas.vue 中的 watch 会自动处理）
}
</script>

<style scoped>
.typeface-panel {
  /* min-height: calc(100vh - 160px); */
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

.level-selector {
  display: flex;
  align-items: center;
  gap: 12px;
}

.label {
  font-size: 14px;
  color: #606266;
  min-width: 80px;
}

.font-size-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 16px;
}

.font-size-item {
  display: flex;
  align-items: center;
  gap: 12px;
}

.font-size-label {
  min-width: 60px;
  font-size: 14px;
  color: #606266;
}

.font-size-unit {
  font-size: 12px;
  color: #909399;
}

.font-tabs {
  margin-top: 0;
}

.font-gallery {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));
  gap: 12px;
  margin-top: 0;
  max-height: 450px;
  overflow-y: auto;
  padding: 4px;
}

.font-chip {
  position: relative;
  border-radius: 8px;
  border: 2px solid #e4e7ed;
  padding: 12px 16px;
  cursor: pointer;
  background: #fff;
  transition: all 0.15s;
  text-align: left;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.font-chip:hover {
  border-color: #409eff;
  box-shadow: 0 2px 8px rgba(64, 158, 255, 0.15);
  transform: translateY(-1px);
}

.font-chip.active {
  border-color: #409eff;
  background: #ecf5ff;
  box-shadow: 0 2px 12px rgba(64, 158, 255, 0.25);
}

.font-name {
  font-size: 14px;
  color: #303133;
  font-weight: 500;
}

.font-active-badge {
  position: absolute;
  top: 4px;
  right: 4px;
  font-size: 10px;
  padding: 2px 6px;
  background: #409eff;
  color: #fff;
  border-radius: 10px;
  font-weight: 500;
}
.fontsize-row {
  display: flex;
  align-items: center;
  gap: 12px;
  margin: 10px 0 0 0;
}
.fontsize-label {font-size:14px;color:#606266;min-width:68px;}
.fontsize-unit {font-size:12px;color:#909399;}
</style>

