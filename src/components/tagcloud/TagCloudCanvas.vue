<template>
  <aside class="tagcloud-panel">
    <header class="panel-head">
      <el-space direction="horizontal" alignment="center" size="small">
        <el-button type="primary" @click="handleRenderCloud">运行生成标签云</el-button>
        <el-dropdown @command="handleExportCommand">
          <el-button>
            导出图片<el-icon style="margin-left:4px"><arrow-down /></el-icon>
          </el-button>
          <template #dropdown>
            <el-dropdown-menu>
              <el-dropdown-item command="svg">导出SVG</el-dropdown-item>
              <el-dropdown-item command="png">导出PNG</el-dropdown-item>
              <el-dropdown-item command="jpeg">导出JPEG</el-dropdown-item>
            </el-dropdown-menu>
          </template>
        </el-dropdown>
        <span style="margin-left: 8px; font-size: 15px;">当前展示的城市数：{{ poiStore.cityOrder.length }}</span>
      </el-space>
    </header>
    <!-- 导出图片设置对话框 -->
    <el-dialog v-model="exportDialogVisible" title="导出图片设置" width="350px" :close-on-click-modal="false">
      <div style="display:flex; gap:10px; align-items:center; margin-bottom:10px;">
        <span style="width:60px;">宽度(px)</span>
        <el-input-number v-model="exportWidth" :min="1" :max="4000" :step="10" size="small" @change="onExportWidthChange" style="width:130px;"/>
      </div>
      <div style="display:flex; gap:10px; align-items:center; margin-bottom:10px;">
        <span style="width:60px;">高度(px)</span>
        <el-input-number v-model="exportHeight" :min="1" :max="4000" :step="10" size="small" @change="onExportHeightChange" style="width:130px;"/>
      </div>
      <div style="display:flex; gap:10px; align-items:center; margin-bottom:10px;">
        <el-checkbox v-model="lockAspectRatio" size="small">锁定比例</el-checkbox>
      </div>
      <template #footer>
        <el-button @click="exportDialogVisible=false">取消</el-button>
        <el-button type="primary" @click="handleExportConfirm">确认导出</el-button>
      </template>
    </el-dialog>
    <div class="canvas-wrapper" ref="wrapperRef">
      <svg ref="svgRef"
           :style="{ background: poiStore.colorSettings.background }"
      ></svg>
      <div v-if="(!poiStore.hasDrawing || !poiStore.cityOrder.length) && !cloudLoading" class="empty-cloud-hint">
        <div class="hint-content">
          <div class="hint-icon">
            <svg viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
              <path d="M12 2L2 7L12 12L22 7L12 2Z" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
              <path d="M2 17L12 22L22 17" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
              <path d="M2 12L12 17L22 12" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
            </svg>
          </div>
          <div class="hint-text">
            <p class="hint-title">准备生成标签云</p>
            <p class="hint-desc">请先在地图上绘制折线，然后点击"运行生成标签云"按钮</p>
          </div>
        </div>
      </div>
      <div v-if="cloudLoading" class="cloud-loading-overlay">
        <div class="cloud-loading-spinner">
          <div class="spinner-dot"></div>
          <div class="spinner-dot"></div>
          <div class="spinner-dot"></div>
          <div class="spinner-dot"></div>
          <div class="spinner-dot"></div>
          <div class="spinner-dot"></div>
          <div class="spinner-dot"></div>
          <div class="spinner-dot"></div>
        </div>
        <span class="cloud-loading-text">请稍等，正在生成标签云...</span>
      </div>
    </div>
  </aside>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, nextTick, watch, computed } from 'vue';
import { usePoiStore } from '@/stores/poiStore';
import * as d3 from 'd3';
import cloud from 'd3-cloud';
import { StripLayout, SpiralLayout, PivotLayout } from '@/utils/treemapLayouts';
import { ElButton, ElSpace, ElDropdown, ElDropdownMenu, ElDropdownItem, ElIcon, ElInputNumber, ElDialog, ElColorPicker, ElCheckbox } from 'element-plus';
import { ArrowDown } from '@element-plus/icons-vue';

const exportDialogVisible = ref(false)
const exportWidth = ref(800)
const exportHeight = ref(600)
const exportFormat = ref('png')
const lockAspectRatio = ref(true);
const origWidth = ref(800);
const origHeight = ref(600);
let _aspectRatio = 1;

const poiStore = usePoiStore();
const wrapperRef = ref(null);
const svgRef = ref(null);
let svg = null;
let graph = null;

// loading 遮罩状态
const cloudLoading = computed(() => poiStore.cloudLoading);

// 优化动态分配字号——对数插值算法
function updateFontSizesForCompiledData(compiledData, fontSettings) {
  const allPoiArr = [];
  Object.values(compiledData).forEach(arr => allPoiArr.push(...arr));
  const rankList = allPoiArr.map(p => parseInt(p.rankInChina)).filter(r => r > 0 && isFinite(r));
  if (!rankList.length) { allPoiArr.forEach(p=>p.size=fontSettings.minFontSize); return; }
  const minRank = Math.min(...rankList);
  const maxRank = Math.max(...rankList);
  if (!isFinite(minRank) || !isFinite(maxRank) || minRank<=0 || maxRank<=0) { allPoiArr.forEach(p=>p.size=fontSettings.minFontSize); return; }
  const logMin = Math.log(minRank);
  const logMax = Math.log(maxRank);
  allPoiArr.forEach(poi => {
    const r = parseInt(poi.rankInChina);
    if (!r || r<=0 || !isFinite(r)) { poi.size=fontSettings.minFontSize; return; }
    const logR = Math.log(r);
    if (logMax-logMin === 0) { poi.size=fontSettings.minFontSize; return; }
    let size = fontSettings.minFontSize + (fontSettings.maxFontSize - fontSettings.minFontSize) * (logMax-logR) / (logMax-logMin);
    poi.size = (isFinite(size) && size>0) ? size : fontSettings.minFontSize;
  });
}

// 计算每个城市景点权重
const calculateAttractionWeights = (data, cityOrder) => {
  return cityOrder.map(city => {
    return {
      city: city,
      attractions: data[city]
        ? data[city].reduce((sum, d) => sum + (d.text.length * d.size), 0)
        : 0
    };
  });
};

// 根据线型类型设置treemap布局
const setupTreemapLayout = (lineType, width, height) => {
  let treemap;

  switch (lineType) {
    case 'Resquarify':
      treemap = d3.treemap()
        .size([width, height])
        .round(true)
        .tile(d3.treemapResquarify);
      break;
    case 'Binary':
      treemap = d3.treemap()
        .size([width, height])
        .round(true)
        .tile(d3.treemapBinary);
      break;
    case 'Dice':
      treemap = d3.treemap()
        .size([width, height])
        .round(true)
        .tile(d3.treemapDice);
      break;
    case 'Slice':
      treemap = d3.treemap()
        .size([width, height])
        .round(true)
        .tile(d3.treemapSlice);
      break;
    case 'Strip':
      treemap = d3.treemap()
        .size([width, height])
        .round(true)
        .tile(StripLayout);
      break;
    case 'Spiral':
      treemap = d3.treemap()
        .size([width, height])
        .round(true)
        .tile(SpiralLayout);
      break;
    case 'Pivot':
      treemap = d3.treemap()
        .size([width, height])
        .round(true)
        .tile(PivotLayout);
      break;
    default:
      treemap = d3.treemap()
        .size([width, height])
        .round(true)
        .tile(d3.treemapResquarify);
      break;
  }

  return treemap;
};

// TreeMap布局算法已从utils/treemapLayouts.js导入

// 构建邻接矩阵
const buildAdjacencyMatrix = (nodes) => {
  const AdjacencyMatrix = [];
  nodes.forEach(a => {
    const temp = [];
    nodes.forEach(b => {
      if (a == b) {
        temp.push(0);
      } else {
        if (areNodesAdjacent(a, b)) {
          temp.push(1);
        } else {
          temp.push(0);
        }
      }
    });
    AdjacencyMatrix.push(temp);
  });
  return AdjacencyMatrix;
};

// 检查节点是否相邻
const areNodesAdjacent = (a, b) => {
  const aLeft = a.x0, aRight = a.x1, aTop = a.y0, aBottom = a.y1;
  const bLeft = b.x0, bRight = b.x1, bTop = b.y0, bBottom = b.y1;

  const adjacentVertically = (((aRight >= bLeft && aLeft <= bRight) || (aLeft >= bLeft && aLeft <= bRight)) && (aTop === bBottom)) ||
    (((bRight >= aLeft && bLeft <= aRight) || (bLeft >= aLeft && bLeft <= aRight)) && (bTop === aBottom));

  const adjacentHorizontally = (((bBottom >= aTop && bTop <= aTop) || (aTop <= bTop && bTop <= aBottom)) && (aRight === bLeft)) ||
    (((aBottom >= bTop && aTop <= bTop) || (bTop <= aTop && aTop <= bBottom)) && (bRight === aLeft));

  return adjacentVertically || adjacentHorizontally;
};

// 为图的每个顶点分配颜色
const graphColoring = (graph, m) => {
  const colorindexs = new Array(graph.length).fill(0);

  function isSafe(vertex, color) {
    for (let i = 0; i < graph.length; i++) {
      if (graph[vertex][i] === 1 && colorindexs[i] === color) {
        return false;
      }
    }
    return true;
  }

  function graphColoringUtil(vertex) {
    if (vertex === graph.length) {
      return true;
    }

    for (let color = 1; color <= m; color++) {
      if (isSafe(vertex, color)) {
        colorindexs[vertex] = color;
        if (graphColoringUtil(vertex + 1)) {
          return true;
        }
        colorindexs[vertex] = 0;
      }
    }
    return false;
  }

  if (!graphColoringUtil(0)) {
    console.log("No solution exists");
    return false;
  }

  return colorindexs;
};

// 绘制折线或曲线路径，可配置样式
const drawSVGLine = (svg, root, cityOrder, lineOpts = {}) => {
  const { lineType = 'curve', width = 2, color = '#aaa' } = lineOpts;

  const recpoints = root.leaves().map(d => {
    return {
      city: d.data.city,
      x: (d.x0 + d.x1) / 2,
      y: (d.y0 + d.y1) / 2
    };
  });
  const orderedPoints = cityOrder
    .map(city => recpoints.find(point => point.city === city))
    .filter(point => point && isFinite(point.x) && isFinite(point.y));
  if (orderedPoints.length < 2) return;
  if(lineType==='none') return;

  let lineGen = d3.line()
    .x(d => d.x)
    .y(d => d.y);
  if(lineType === 'curve') {
    lineGen.curve(d3.curveCatmullRom.alpha(0.5));
  } else if(lineType === 'polyline') {
    lineGen.curve(d3.curveLinear);
  }
  svg.append("path")
    .datum(orderedPoints)
    .attr("fill", "none")
    .attr("stroke", color)
    .attr("stroke-width", width)
    .attr("d", lineGen);
};

// 绘制单个词云
const drawWordCloud = (i, svg, cities, city, x, y, colorIndex, color, width, height, resolve) => {
  if (!isFinite(x) || !isFinite(y)) { if(resolve)resolve(); return; }
  const words = cities[city] ? cities[city].map(d => ({ text: d.text, size: d.size, color: color })) : [];
  if (words.length == 0) { if (resolve) resolve(); return; }
  words.push({ text: city, size: 46, isCity: true, colorIndex: -1 });
  const layout = cloud()
    .size([width, height])
    .words(words)
    .padding(1)
    .rotate(() => 0)
    .font(poiStore.fontSettings.fontFamily || "Arial")
    .fontSize(d => d.size)
    .spiral("archimedean")
    .random(() => 0.5)
    .on("end", words => {
      const g = svg.append("g")
        .attr("transform", `translate(${x},${y})`)
        .attr("data-colorindex", colorIndex)
        .attr("id", `cloud-${i}`);
      g.selectAll("text")
        .data(words)
        .enter().append("text")
        .attr("class", d => d.isCity ? "city-label" : "word")
        .style("font-size", d => d.size + "px")
        .style("font-family", poiStore.fontSettings.fontFamily || "Arial")
        .style("font-weight", poiStore.fontSettings.fontWeight || "700")
        .style("fill", d => d.isCity ? "red" : d.color)
        .attr("text-anchor", "middle")
        .attr("transform", d => `translate(${d.x},${d.y})rotate(${d.rotate})`)
        .text(d => d.text);
      if (resolve) resolve();
    });
  layout.start();
};

// 绘制所有的词云
const drawAllWordClouds = async (svg, data, cityOrder, width, height, lineType) => {
  const cityData = calculateAttractionWeights(data, cityOrder);
  
  // 构建层次结构 - 原项目使用values作为children
  const rootData = {
    values: cityData
  };
  
  const root = d3.hierarchy(rootData, function (d) { 
    return d.values; 
  }).sum(function (d) { 
    return d.attractions || 0; 
  });

  // 应用TreeMap布局
  const treemap = setupTreemapLayout(lineType, width, height);
  treemap(root);

  const leaves = root.leaves();
  
  // 将耗时计算分批进行，让浏览器有机会渲染动画
  // 使用 setTimeout 让浏览器在计算间隙渲染
  await new Promise(resolve => setTimeout(resolve, 0));
  graph = buildAdjacencyMatrix(leaves);
  
  await new Promise(resolve => setTimeout(resolve, 0));
  const colorindexs = graphColoring(graph, poiStore.colorNum);

  drawSVGLine(svg, root, cityOrder, {
    lineType: poiStore.linePanel?.type,
    width: poiStore.linePanel?.width,
    color: poiStore.linePanel?.color
  });

  const cityInfo = {};
  leaves.forEach(d => {
    const city = d.data.city;
    const x0 = d.x0;
    const y0 = d.y0;
    const x1 = d.x1;
    const y1 = d.y1;
    const centerX = (x0 + x1) / 2;
    const centerY = (y0 + y1) / 2;
    const cloudWidth = x1 - x0;
    const cloudHeight = y1 - y0;
    cityInfo[city] = { centerX, centerY, width: cloudWidth, height: cloudHeight };
  });

  // 使用Promise等待所有词云绘制完成
  const cloudPromises = [];
  cityOrder.forEach((city, i) => {
    const tempInfo = cityInfo[city];
    if (!tempInfo) return;
    
    const cloudWidth = tempInfo.width;
    const cloudHeight = tempInfo.height;
    const centerX = tempInfo.centerX;
    const centerY = tempInfo.centerY;
    const colorIndex = colorindexs[i] - 1;
    const color = poiStore.Colors[colorIndex % poiStore.Colors.length];
    
    const promise = new Promise((resolve) => {
      drawWordCloud(i, svg, data, city, centerX, centerY, colorIndex, color, cloudWidth, cloudHeight, resolve);
    });
    cloudPromises.push(promise);
  });

  return Promise.all(cloudPromises).then(() => {
    console.log('所有词云绘制完成');
  });
};

const handleRenderCloud = async () => {
  poiStore.setCloudLoading(true);
  // 确保 DOM 更新，让 loading overlay 显示
  await nextTick();
  // 使用 requestAnimationFrame 确保浏览器至少渲染一次 loading overlay
  await new Promise(resolve => requestAnimationFrame(resolve));
  // 再等待一帧，确保 loading overlay 完全渲染
  await new Promise(resolve => requestAnimationFrame(resolve));
  
  const startTime = Date.now();
  const minDisplayTime = 300; // 最小显示时间 300ms，确保用户能看到 loading
  
  try {
    console.info('[TagCloudCanvas] handleRenderCloud 开始', {
      hasDrawing: poiStore.hasDrawing,
      cityOrderCount: poiStore.cityOrder.length,
      compiledKeys: Object.keys(poiStore.compiledData || {}).length,
    });
    if (!poiStore.hasDrawing) {
      console.warn('请先在地图上绘制折线');
      return;
    }
    // 检查数据是否已准备好
    if (!poiStore.cityOrder.length || !Object.keys(poiStore.compiledData).length) {
      console.warn('数据未准备好，请等待数据处理完成', {
        cityOrder: poiStore.cityOrder,
        compiledDataKeys: Object.keys(poiStore.compiledData || {}),
      });
      return;
    }
    await nextTick();
    if (!svgRef.value || !wrapperRef.value) return;
    const rect = wrapperRef.value.getBoundingClientRect();
    const width = Math.floor(rect.width);
    const height = Math.floor(rect.height);
    svg = d3.select(svgRef.value);
    svg.selectAll("*").remove();
    svg.attr("width", width).attr("height", height);
    const data = poiStore.compiledData;
    const cityOrder = poiStore.cityOrder;
    const lineType = poiStore.lineType || 'Pivot';
    // 字号分配
    updateFontSizesForCompiledData(data, poiStore.fontSettings);
    await nextTick();
    // 再次让浏览器渲染一次，确保 loading overlay 可见
    await new Promise(resolve => setTimeout(resolve, 50));
    // 等待所有词云绘制完成后再关闭 loading
    await drawAllWordClouds(svg, data, cityOrder, width, height, lineType);
  } finally {
    // 确保 loading 至少显示最小时间
    const elapsed = Date.now() - startTime;
    if (elapsed < minDisplayTime) {
      await new Promise(resolve => setTimeout(resolve, minDisplayTime - elapsed));
    }
    poiStore.setCloudLoading(false);
  }
};

const handleExportCommand = (command) => {
  if(command === 'svg') {
    exportAsSVG();
  } else if(command === 'png' || command==='jpeg') {
    prepareExportDialog(command)
  }
};

function prepareExportDialog(format) {
  exportFormat.value = format;
  // 尺寸默认用svg实际宽高
  if(svgRef.value) {
    const svg = svgRef.value;
    const rect = svg.getBBox();
    const w = Math.round(rect.width || svg.width?.baseVal?.value || 800);
    const h = Math.round(rect.height || svg.height?.baseVal?.value || 600);
    exportWidth.value = w;
    exportHeight.value = h;
    origWidth.value = w;
    origHeight.value = h;
    _aspectRatio = w/h;
  } else {
    exportWidth.value = 800;
    exportHeight.value = 600;
    origWidth.value = 800;
    origHeight.value = 600;
    _aspectRatio = 800 / 600;
  }
  lockAspectRatio.value = true;
  exportDialogVisible.value = true;
}

// 响应宽度、锁定比例
function onExportWidthChange(val) {
  if (lockAspectRatio.value && origWidth.value && origHeight.value) {
    const w = Number(val)||1;
    exportHeight.value = Math.round(w/origWidth.value*origHeight.value);
  }
}
function onExportHeightChange(val) {
  if (lockAspectRatio.value && origWidth.value && origHeight.value) {
    const h = Number(val)||1;
    exportWidth.value = Math.round(h/origHeight.value*origWidth.value);
  }
}

const handleExportConfirm = () => {
  exportDialogVisible.value = false;
  exportAsRaster(
    exportFormat.value,
    exportWidth.value,
    exportHeight.value,
    '#ffffff' // 始终使用默认白底
  );
}

const exportAsSVG = () => {
  if (!svgRef.value) return;
  const svgElement = svgRef.value;
  const svgString = new XMLSerializer().serializeToString(svgElement);
  const blob = new Blob([svgString], { type: 'image/svg+xml' });
  const url = URL.createObjectURL(blob);
  const link = document.createElement('a');
  link.href = url;
  link.download = 'tag-cloud-treemap.svg';
  link.click();
  URL.revokeObjectURL(url);
};

// SVG转图片格式加强，支持尺寸和背景色
const exportAsRaster = async (format = 'png', exportWidth=800, exportHeight=600, bgColor='#ffffff') => {
  if (!svgRef.value) return;
  const svgElement = svgRef.value;
  const serializer = new XMLSerializer();
  let svgString = serializer.serializeToString(svgElement);
  if(!svgString.match(/xmlns="http:\/\/www.w3.org\/2000\/svg"/)){
    svgString = svgString.replace('<svg', '<svg xmlns="http://www.w3.org/2000/svg"');
  }
  const svg64 = btoa(unescape(encodeURIComponent(svgString)));
  const imageSrc = `data:image/svg+xml;base64,${svg64}`;
  const img = new window.Image();
  img.onload = function() {
    const canvas = document.createElement('canvas');
    canvas.width = exportWidth;
    canvas.height = exportHeight;
    const ctx = canvas.getContext('2d');
    ctx.save();
    ctx.fillStyle = bgColor;
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    ctx.restore();
    ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
    const type = format === 'jpeg' ? 'image/jpeg' : 'image/png';
    const link = document.createElement('a');
    link.href = canvas.toDataURL(type);
    link.download = `tag-cloud-treemap.${format}`;
    link.click();
  };
  img.onerror = () => {
    alert('图片导出失败，请重试！');
  };
  img.src = imageSrc;
};

onMounted(() => {
  // 初始化SVG
  if (svgRef.value && wrapperRef.value) {
    const rect = wrapperRef.value.getBoundingClientRect();
    svg = d3.select(svgRef.value);
    svg.attr("width", rect.width).attr("height", rect.height);
  }
  // 新增：监听配色变化事件
  window.__refreshTagCloudListener__ = () => {
    if (poiStore.hasDrawing) {
      handleRenderCloud();
    }
  };
  window.addEventListener('refreshTagCloud', window.__refreshTagCloudListener__);
});

onBeforeUnmount(() => {
  window.removeEventListener('refreshTagCloud', window.__refreshTagCloudListener__);
  delete window.__refreshTagCloudListener__;
});

watch(
  () => poiStore.hasDrawing,
  (hasDrawing) => {
    if (!hasDrawing && svg) {
      svg.selectAll("*").remove();
    }
  }
);

watch(
  () => poiStore.cityOrder.length,
  (length) => {
    console.info('[TagCloudCanvas] cityOrder 长度', length);
  }
);

watch(
  () => Object.keys(poiStore.compiledData || {}).length,
  (length) => {
    console.info('[TagCloudCanvas] compiledData 键数量', length);
  }
);

// watch字体设置深度变化，自动刷新并显示loading
watch(
  () => poiStore.fontSettings,
  (newVal, oldVal) => {
    if (poiStore.hasDrawing) {
      handleRenderCloud();
    }
  },
  { deep: true }
);

// watch布局类型变化，自动刷新并显示loading
watch(
  () => poiStore.lineType,
  (newVal, oldVal) => {
    if (poiStore.hasDrawing && oldVal !== undefined) {
      handleRenderCloud();
    }
  }
);

// setup标签云watch监听linePanel配置
watch(
  () => ({...poiStore.linePanel}),
  (val) => {
    if (poiStore.hasDrawing) {
      handleRenderCloud();
    }
  },
  { deep: true }
);
</script>

<style scoped>
.tagcloud-panel {
  background:rgb(247,249,252);
  color: #000000;
  padding: 24px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  min-width: 650px;
  width: 100%;
  height: 100%;
  box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.05);
  overflow: hidden;
}

.panel-head {
  display: flex;
  align-items: center;
  justify-content: flex-start;
  margin-bottom: 16px;
  flex-shrink: 0;
}

.canvas-wrapper {
  flex: 1;
  display: flex;
  align-items: stretch;
  justify-content: stretch;
  width: 100%;
  height: 100%;
  min-height: 0;
  overflow: hidden;
  position: relative;
}

svg {
  width: 100%;
  height: 100%;
  display: block;
}

.empty-cloud-hint {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(255, 255, 255, 0);
  backdrop-filter: blur(8px);
  z-index: 5;
  pointer-events: none;
}

.hint-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 24px;
  padding: 40px;
  text-align: center;
  max-width: 500px;
}

.hint-icon {
  width: 80px;
  height: 80px;
  color: rgba(104, 104, 104, 0.549);
  animation: float 3s ease-in-out infinite;
}

@keyframes float {
  0%, 100% {
    transform: translateY(0px);
  }
  50% {
    transform: translateY(-10px);
  }
}

.hint-text {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.hint-title {
  margin: 0;
  font-size: 20px;
  font-weight: 600;
  color: rgba(104, 104, 104, 0.549);
  letter-spacing: 0.5px;
}

.hint-desc {
  margin: 0;
  font-size: 14px;
  line-height: 1.6;
  color: rgba(104, 104, 104, 0.549);
  letter-spacing: 0.3px;
}
.cloud-loading-overlay {
  position: absolute;
  z-index: 999;
  top: 0; left: 0; width: 100%; height: 100%;
  background: rgba(255, 255, 255, 0.85);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: #333;
  font-size: 20px;
  pointer-events: all;
  gap: 18px;
  font-weight: 500;
  backdrop-filter: blur(8px);
}

.cloud-loading-spinner {
  position: relative;
  width: 60px;
  height: 60px;
  margin-bottom: 12px;
  will-change: transform;
}

.spinner-dot {
  position: absolute;
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #409eff;
  top: 0;
  left: 50%;
  transform-origin: 0 30px;
  will-change: transform, opacity;
  animation: spinner-rotate 1.2s linear infinite;
}

.spinner-dot:nth-child(1) {
  animation-delay: 0s;
  opacity: 1;
}

.spinner-dot:nth-child(2) {
  animation-delay: 0.15s;
  opacity: 0.875;
}

.spinner-dot:nth-child(3) {
  animation-delay: 0.3s;
  opacity: 0.75;
}

.spinner-dot:nth-child(4) {
  animation-delay: 0.45s;
  opacity: 0.625;
}

.spinner-dot:nth-child(5) {
  animation-delay: 0.6s;
  opacity: 0.5;
}

.spinner-dot:nth-child(6) {
  animation-delay: 0.75s;
  opacity: 0.375;
}

.spinner-dot:nth-child(7) {
  animation-delay: 0.9s;
  opacity: 0.25;
}

.spinner-dot:nth-child(8) {
  animation-delay: 1.05s;
  opacity: 0.125;
}

@keyframes spinner-rotate {
  to {
    transform: rotate(360deg);
  }
}

.cloud-loading-text {
  margin-top: 8px;
  color: #606266;
  font-size: 16px;
}
</style>

