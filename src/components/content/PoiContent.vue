<template>
  <section class="panel-card content-panel">
    <div class="work-grid">
      <div class="map-container">
        <PoiMap />
      </div>
      <!-- table-card留空，后续添加其他内容 -->
      <div class="table-card-placeholder"></div>
    </div>
  </section>
</template>

<script setup>
import { onMounted } from 'vue';
import { usePoiStore } from '@/stores/poiStore';
import PoiMap from './PoiMap.vue';

const poiStore = usePoiStore();

// POI数据加载已移至PoiMap组件的loadMap函数中，避免重复加载
</script>

<style scoped>
.content-panel {
  height: 100%;
  display: flex;
  flex-direction: column;
  min-height: 0;
  overflow: hidden;
}

.work-grid {
  display: grid;
  grid-template-rows: 1fr 1fr;
  gap: 20px;
  flex: 1 1 auto;
  min-height: 0;
  height: 100%;
  overflow: hidden;
}

.work-grid > * {
  min-height: 0;
  overflow: hidden;
  height: 100%;
}

.map-container {
  min-height: 0;
  overflow: hidden;
  height: 100%;
  display: flex;
  flex-direction: column;
}

.table-card-placeholder {
  background: #fff;
  border-radius: 12px;
  border: 1px dashed #dcdfe6;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #909399;
  font-size: 14px;
}
</style>

