<template>
  <aside class="tagcloud-panel">
    <header class="panel-head">
      <el-space direction="horizontal" alignment="center" size="small">
        <el-button
          type="primary"
          data-intro-target="runTagCloudBtn"
          @click="handleRenderCloud"
        >
          运行生成标签云
        </el-button>
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
           :style="{ background: poiStore.colorSettings.backgroundMode === 'single' ? poiStore.colorSettings.background : 'transparent' }"
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
import introJs from 'intro.js';
import 'intro.js/minified/introjs.min.css';
import { usePoiStore } from '@/stores/poiStore';
import * as d3 from 'd3';
import cloud from 'd3-cloud';
import { StripLayout, SpiralLayout, PivotLayout } from '@/utils/treemapLayouts';
import { cityNameToPinyin } from '@/utils/cityNameToPinyin';
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
// 保存当前的布局信息，用于只更新路径
let currentRoot = null;
let currentCityOrder = null;
let currentLayoutType = null;
let currentColorIndexs = null; // 保存当前的颜色索引，用于更新背景
let secondIntroStarted = false;

// loading 遮罩状态
const cloudLoading = computed(() => poiStore.cloudLoading);

const getDrawLineButtonElement = () => {
  return (
    document.querySelector('[data-intro-target="drawLineTrigger"]') ||
    document.querySelector('.map-head .dropdown-btn')
  );
};

const getMapCanvasElement = () => {
  return (
    document.querySelector('[data-intro-target="mapCanvas"]') ||
    document.querySelector('.map-canvas') ||
    document.querySelector('.map-wrapper')
  );
};

const getRunTagCloudButtonElement = () => {
  return (
    document.querySelector('[data-intro-target="runTagCloudBtn"]') ||
    document.querySelector('.tagcloud-panel .panel-head .el-button--primary')
  );
};

const startDrawGuideIntro = () => {
  if (secondIntroStarted) return;
  secondIntroStarted = true;

  const attemptStart = (retries = 8) => {
    const drawBtn = getDrawLineButtonElement();
    const mapCanvas = getMapCanvasElement();
    const runBtn = getRunTagCloudButtonElement();

    if (drawBtn && mapCanvas && runBtn) {
      try {
        const intro = introJs.tour();
        intro.addSteps([
          {
            element: drawBtn,
            intro:
              '<div style="line-height:1.6;"><strong style="font-size:16px;color:#1f2333;">绘制折线</strong><br/><span style="color:#64748b;">点击此处选择“手绘折线”或“自定义始末点”，划定需要分析的路线。</span></div>',
          },
          {
            element: mapCanvas,
            intro:
              '<div style="line-height:1.6;"><strong style="font-size:16px;color:#1f2333;">地图区域</strong><br/><span style="color:#64748b;">在地图上完成折线绘制，系统会根据路线经过的城市准备标签数据。</span></div>',
          },
          {
            element: runBtn,
            intro:
              '<div style="line-height:1.6;"><strong style="font-size:16px;color:#1f2333;">运行生成标签云</strong><br/><span style="color:#64748b;">绘制完成后，再次点击该按钮即可生成路线对应的标签云。</span></div>',
          },
        ]);
        intro.setOptions({
          nextLabel: '下一步 →',
          prevLabel: '← 上一步',
          skipLabel: '跳过',
          doneLabel: '完成',
          showStepNumbers: true,
          showProgress: true,
          disableInteraction: false,
          tooltipClass: 'customTooltipClass',
          highlightClass: 'customHighlightClass',
          exitOnOverlayClick: true,
          exitOnEsc: true,
          keyboardNavigation: true,
          tooltipRenderAsHtml: true,
        });
        intro.onComplete(() => {
          secondIntroStarted = false;
        });
        intro.onExit(() => {
          secondIntroStarted = false;
        });
        intro.start();
      } catch (error) {
        console.error('[TagCloudCanvas] 二次引导启动失败', error);
        secondIntroStarted = false;
      }
      return;
    }

    if (retries > 0) {
      setTimeout(() => attemptStart(retries - 1), 200);
    } else {
      console.warn('[TagCloudCanvas] 未找到绘制引导元素');
      secondIntroStarted = false;
    }
  };

  nextTick(() => {
    setTimeout(() => attemptStart(), 120);
  });
};

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
  if(lineType==='none') {
    // 如果类型为none，删除已存在的路径
    svg.select("#city-route-path").remove();
    return;
  }

  let lineGen = d3.line()
    .x(d => d.x)
    .y(d => d.y);
  if(lineType === 'curve') {
    lineGen.curve(d3.curveCatmullRom.alpha(0.5));
  } else if(lineType === 'polyline') {
    lineGen.curve(d3.curveLinear);
  }
  
  // 先删除已存在的路径（如果存在）
  svg.select("#city-route-path").remove();
  
  // 创建新路径，添加特定ID以便后续更新
  // 确保路径在 SVG 的最前面（作为第一个子元素），这样词云会覆盖在路径上方
  const svgNode = svg.node();
  const firstChild = svgNode?.firstChild;
  
  // 使用 insert 方法，将路径插入到第一个元素之前（如果存在），否则插入到最前面
  const path = svg.insert("path", firstChild ? () => firstChild : null)
    .attr("id", "city-route-path")
    .datum(orderedPoints)
    .attr("fill", "none")
    .attr("stroke", color)
    .attr("stroke-width", width)
    .attr("d", lineGen);
  
  // 双重保险：确保路径是第一个子元素
  const pathNode = path.node();
  if (svgNode && pathNode && svgNode.firstChild !== pathNode) {
    svgNode.insertBefore(pathNode, svgNode.firstChild);
  }
};

// 只更新路径样式，不重新绘制整个标签云
const updatePathOnly = () => {
  if (!svg || !currentRoot || !currentCityOrder) return;
  
  const lineOpts = {
    lineType: poiStore.linePanel?.type || 'curve',
    width: poiStore.linePanel?.width || 2,
    color: poiStore.linePanel?.color || '#aaa'
  };
  
  drawSVGLine(svg, currentRoot, currentCityOrder, lineOpts);
};

// 只更新字体样式，不重新绘制整个标签云
const updateFontStylesOnly = () => {
  if (!svg) return;
  
  const fontFamily = poiStore.fontSettings.fontFamily || "Arial";
  const fontWeight = poiStore.fontSettings.fontWeight || "700";
  
  // 更新所有文本元素的字体样式
  svg.selectAll("text.word, text.city-label")
    .style("font-family", fontFamily)
    .style("font-weight", fontWeight);
};

// 只更新颜色，不重新绘制整个标签云
const updateColorsOnly = () => {
  if (!svg) return;
  
  const textColorMode = poiStore.colorSettings.textColorMode || 'multi';
  const textSingleColor = poiStore.colorSettings.textSingleColor || 'rgb(0, 0, 0)';
  
  // 更新背景矩形（如果背景模式是复色）
  if (poiStore.colorSettings.backgroundMode === 'multi') {
    const opacity = poiStore.colorSettings.backgroundMultiColorOpacity ?? 0.1;
    const leaves = currentRoot?.leaves() || [];
    const cityOrder = currentCityOrder || [];
    const colorindexs = currentColorIndexs;
    
    if (leaves.length > 0 && cityOrder.length > 0 && colorindexs) {
      svg.selectAll("rect.treemap-background").remove();
      
      leaves.forEach((d, i) => {
        const city = d.data.city;
        const cityIndex = cityOrder.indexOf(city);
        if (cityIndex === -1) return;
        
        const colorIndex = colorindexs[cityIndex] - 1;
        const bgColor = poiStore.Colors[colorIndex % poiStore.Colors.length];
        
        // 将颜色转换为rgba格式以支持透明度
        let rgbaColor = bgColor;
        if (bgColor.startsWith('#')) {
          const hex = bgColor.slice(1);
          const r = parseInt(hex.slice(0, 2), 16);
          const g = parseInt(hex.slice(2, 4), 16);
          const b = parseInt(hex.slice(4, 6), 16);
          rgbaColor = `rgba(${r}, ${g}, ${b}, ${opacity})`;
        } else if (bgColor.startsWith('rgb')) {
          const rgbMatch = bgColor.match(/\d+/g);
          if (rgbMatch && rgbMatch.length >= 3) {
            rgbaColor = `rgba(${rgbMatch[0]}, ${rgbMatch[1]}, ${rgbMatch[2]}, ${opacity})`;
          }
        }
        
        svg.insert("rect", () => svg.select("#city-route-path").node()?.nextSibling || null)
          .attr("x", d.x0)
          .attr("y", d.y0)
          .attr("width", d.x1 - d.x0)
          .attr("height", d.y1 - d.y0)
          .attr("fill", rgbaColor)
          .attr("stroke", "none")
          .attr("class", "treemap-background");
      });
    }
  } else {
    // 如果背景模式是单色，移除所有背景矩形
    svg.selectAll("rect.treemap-background").remove();
  }
  
  // 遍历所有词云 group，根据 colorIndex 更新颜色
  svg.selectAll("g[data-colorindex]").each(function() {
    const g = d3.select(this);
    const colorIndexAttr = g.attr("data-colorindex");
    const colorIndex = parseInt(colorIndexAttr);
    
    // 城市标签保持红色
    g.selectAll("text.city-label")
      .style("fill", "red");
    
    // 如果 colorIndex 无效，跳过
    if (isNaN(colorIndex) || colorIndex < 0) {
      return;
    }
    
    // 根据文字配色模式更新颜色
    if (textColorMode === 'single') {
      // 单色模式：所有文字使用统一颜色
      g.selectAll("text.word")
        .style("fill", textSingleColor);
    } else {
      // 复色模式：使用原来的颜色逻辑
      const newColor = poiStore.Colors[colorIndex % poiStore.Colors.length];
      g.selectAll("text.word")
        .style("fill", newColor);
    }
  });
};

// 绘制单个词云
const drawWordCloud = (i, svg, cities, city, x, y, colorIndex, color, width, height, resolve) => {
  if (!isFinite(x) || !isFinite(y)) { if(resolve)resolve(); return; }
  const words = cities[city] ? cities[city].map(d => ({ text: d.text, size: d.size, color: color })) : [];
  if (words.length == 0) { if (resolve) resolve(); return; }
  // 根据语言设置选择城市名：中文使用原城市名，英文使用拼音
  let cityName = poiStore.fontSettings.language === 'en' 
    ? cityNameToPinyin(city) 
    : city;
  // 如果需要显示序号，在城市名前添加序号（序号从1开始）
  if (poiStore.fontSettings.showCityIndex) {
    cityName = `${i + 1}. ${cityName}`;
  }
  words.push({ text: cityName, size: 46, isCity: true, colorIndex: -1 });
  
  // 判断文字配色模式
  const textColorMode = poiStore.colorSettings.textColorMode || 'multi';
  const textSingleColor = poiStore.colorSettings.textSingleColor || 'rgb(0, 0, 0)';
  
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
        .style("fill", d => {
          // 城市标签保持红色
          if (d.isCity) return "red";
          // 文字配色模式：单色使用统一颜色，复色使用原来的颜色
          if (textColorMode === 'single') {
            return textSingleColor;
          } else {
            return d.color;
          }
        })
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

  // 保存当前布局信息，用于后续只更新路径
  currentRoot = root;
  currentCityOrder = cityOrder;
  currentLayoutType = lineType;
  currentColorIndexs = colorindexs; // 保存颜色索引

  drawSVGLine(svg, root, cityOrder, {
    lineType: poiStore.linePanel?.type,
    width: poiStore.linePanel?.width,
    color: poiStore.linePanel?.color
  });

  // 如果背景模式是复色，绘制背景矩形
  if (poiStore.colorSettings.backgroundMode === 'multi') {
    const opacity = poiStore.colorSettings.backgroundMultiColorOpacity ?? 0.1;
    leaves.forEach((d, i) => {
      const city = d.data.city;
      const cityIndex = cityOrder.indexOf(city);
      if (cityIndex === -1) return;
      
      const colorIndex = colorindexs[cityIndex] - 1;
      const bgColor = poiStore.Colors[colorIndex % poiStore.Colors.length];
      
      // 将颜色转换为rgba格式以支持透明度
      let rgbaColor = bgColor;
      if (bgColor.startsWith('#')) {
        const hex = bgColor.slice(1);
        const r = parseInt(hex.slice(0, 2), 16);
        const g = parseInt(hex.slice(2, 4), 16);
        const b = parseInt(hex.slice(4, 6), 16);
        rgbaColor = `rgba(${r}, ${g}, ${b}, ${opacity})`;
      } else if (bgColor.startsWith('rgb')) {
        // 如果是rgb格式，转换为rgba
        const rgbMatch = bgColor.match(/\d+/g);
        if (rgbMatch && rgbMatch.length >= 3) {
          rgbaColor = `rgba(${rgbMatch[0]}, ${rgbMatch[1]}, ${rgbMatch[2]}, ${opacity})`;
        }
      }
      
      // 绘制背景矩形，确保在路径之后、词云之前
      svg.insert("rect", () => svg.select("#city-route-path").node()?.nextSibling || null)
        .attr("x", d.x0)
        .attr("y", d.y0)
        .attr("width", d.x1 - d.x0)
        .attr("height", d.y1 - d.y0)
        .attr("fill", rgbaColor)
        .attr("stroke", "none")
        .attr("class", "treemap-background");
    });
  }

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
  if (!poiStore.hasDrawing) {
    startDrawGuideIntro();
    return;
  }

  poiStore.setCloudLoading(true);
  // 确保 DOM 更新，让 loading overlay 显示
  await nextTick();
  // 使用 requestAnimationFrame 确保浏览器至少渲染一次 loading overlay
  await new Promise(resolve => requestAnimationFrame(resolve));
  // 再等待一帧，确保 loading overlay 完全渲染
  await new Promise(resolve => requestAnimationFrame(resolve));
  
  const startTime = Date.now();
  const minDisplayTime = 10; // 最小显示时间 300ms，确保用户能看到 loading
  
  try {
    console.info('[TagCloudCanvas] handleRenderCloud 开始', {
      hasDrawing: poiStore.hasDrawing,
      cityOrderCount: poiStore.cityOrder.length,
      compiledKeys: Object.keys(poiStore.compiledData || {}).length,
    });
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
    // 清空保存的布局信息
    currentRoot = null;
    currentCityOrder = null;
    currentLayoutType = null;
    currentColorIndexs = null;
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
  // 新增：监听配色变化事件 - 只更新颜色，不重绘整个标签云
  window.__refreshTagCloudListener__ = () => {
    if (poiStore.hasDrawing && svg) {
      // 如果已经有绘制好的标签云，只更新颜色
      updateColorsOnly();
    } else if (poiStore.hasDrawing) {
      // 如果还没有绘制，需要完整渲染
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
      // 清空保存的布局信息
      currentRoot = null;
      currentCityOrder = null;
      currentLayoutType = null;
      currentColorIndexs = null;
      currentColorIndexs = null;
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

// 中文字体列表（用于判断字体类型）
const chineseFonts = [
  '等线', '等线 Light', '方正舒体', '方正姚体', '仿宋', '黑体',
  '华文彩云', '华文仿宋', '华文琥珀', '华文楷体', '华文隶书', '华文宋体', 
  '华文细黑', '华文新魏', '华文行楷', '华文中宋', '楷体', '隶书', 
  '宋体', '微软雅黑', '微软雅黑 Light', '新宋体', '幼圆', '思源黑体'
];

// 英文字体列表（用于判断字体类型）
const englishFonts = [
  'Arial', 'Inter', 
  'Times New Roman', 'Georgia', 'Verdana', 'Courier New', 'Comic Sans MS',
  'Impact', 'Trebuchet MS', 'Palatino', 'Garamond', 
  'Helvetica', 'Tahoma', 'Lucida Console', 'Century Gothic', 'Franklin Gothic',
  'Baskerville',
];

// 判断字体是否为中文字体
const isChineseFont = (fontName) => {
  return chineseFonts.includes(fontName);
};

// 判断字体是否为英文字体
const isEnglishFont = (fontName) => {
  return englishFonts.includes(fontName);
};

// watch字体设置深度变化 - 区分字号变化和字体样式变化
watch(
  () => poiStore.fontSettings,
  (newVal, oldVal) => {
    if (!oldVal) {
      // 首次初始化，需要完整渲染
      if (poiStore.hasDrawing) {
        handleRenderCloud();
      }
      return;
    }
    
    // 检查是否是语言变化
    const isLanguageChanged = newVal.language !== oldVal.language;
    
    // 检查是否是字号变化（minFontSize 或 maxFontSize）
    const isFontSizeChanged = 
      newVal.minFontSize !== oldVal.minFontSize || 
      newVal.maxFontSize !== oldVal.maxFontSize;
    
    // 检查是否是字体变化（fontFamily）
    const isFontFamilyChanged = newVal.fontFamily !== oldVal.fontFamily;
    
    // 检查是否是字重变化（fontWeight）
    const isFontWeightChanged = newVal.fontWeight !== oldVal.fontWeight;
    
    // 检查是否是序号显示变化（showCityIndex）
    const isShowCityIndexChanged = newVal.showCityIndex !== oldVal.showCityIndex;
    
    if (isLanguageChanged) {
      // 语言变化需要重新编译数据并完整重绘
      // PoiMap.vue 中的 watch 会自动重新编译数据
      // 这里等待数据编译完成后再重绘
      if (poiStore.hasDrawing) {
        // 使用 nextTick 确保数据已经重新编译
        nextTick(() => {
          setTimeout(() => {
            handleRenderCloud();
          }, 100); // 给一点时间让数据编译完成
        });
      }
    } else if (isShowCityIndexChanged) {
      // 序号显示变化需要重新计算布局（因为序号会影响标签文本长度），需要完整重绘
      if (poiStore.hasDrawing) {
        handleRenderCloud();
      }
    } else if (isFontSizeChanged) {
      // 字号变化需要重新计算字号分配和布局，需要完整重绘
      if (poiStore.hasDrawing) {
        handleRenderCloud();
      }
    } else if (isFontFamilyChanged) {
      // 字体变化：需要根据字体类型决定处理方式
      const newFontIsChinese = isChineseFont(newVal.fontFamily);
      const oldFontIsChinese = isChineseFont(oldVal.fontFamily);
      
      if (newFontIsChinese && oldFontIsChinese) {
        // 中文字体之间的切换：只需要更新样式，不重绘
        if (poiStore.hasDrawing && svg) {
          updateFontStylesOnly();
        }
      } else if (isEnglishFont(newVal.fontFamily) || isEnglishFont(oldVal.fontFamily)) {
        // 涉及英文字体的变化：需要完整重绘（因为英文字体变化可能导致标签大小变化）
        if (poiStore.hasDrawing) {
          handleRenderCloud();
        }
      } else {
        // 其他情况（未知字体）：为了安全起见，完整重绘
        if (poiStore.hasDrawing) {
          handleRenderCloud();
        }
      }
    } else if (isFontWeightChanged && poiStore.hasDrawing && svg) {
      // 字重变化：只需要更新样式，不重绘
      updateFontStylesOnly();
    } else if (poiStore.hasDrawing) {
      // 其他情况，需要完整渲染
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

// setup标签云watch监听linePanel配置 - 只更新路径，不重绘整个标签云
watch(
  () => ({...poiStore.linePanel}),
  (val) => {
    if (poiStore.hasDrawing && currentRoot && currentCityOrder) {
      // 如果已经有绘制好的标签云，只更新路径元素
      updatePathOnly();
    } else if (poiStore.hasDrawing) {
      // 如果还没有绘制，需要完整渲染
      handleRenderCloud();
    }
  },
  { deep: true }
);

// watch配色设置变化 - 只更新颜色，不重绘整个标签云
watch(
  () => ({...poiStore.colorSettings}),
  (newVal, oldVal) => {
    if (!oldVal) {
      // 首次初始化，需要完整渲染
      if (poiStore.hasDrawing) {
        handleRenderCloud();
      }
      return;
    }
    
    if (poiStore.hasDrawing && svg) {
      // 如果已经有绘制好的标签云，只更新颜色
      updateColorsOnly();
    } else if (poiStore.hasDrawing) {
      // 如果还没有绘制，需要完整渲染
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

