<template>
  <div class="batch-test-panel">
    <div class="panel-header">
      <h2>批量测试</h2>
    </div>
    
    <el-tabs v-model="activeTab" class="test-tabs">
      <!-- 手动输入模式 -->
      <el-tab-pane label="手动输入" name="manual">
        <div class="tab-content">
          <div class="input-section">
            <label>城市节点（用逗号或换行分隔）：</label>
            <el-input
              v-model="manualCityInput"
              type="textarea"
              :rows="4"
              placeholder="例如：北京,上海,广州,深圳 或 北京，上海，广州，深圳"
              class="city-input"
            />
            <el-button 
              type="primary" 
              @click="handleManualInput"
              :disabled="!manualCityInput.trim()"
              style="margin-top: 12px;"
            >
              添加为测试用例
            </el-button>
          </div>
          
          <div v-if="manualTestCases.length > 0" class="test-cases-section">
            <h3>已添加的测试用例（{{ manualTestCases.length }}个）：</h3>
            <div class="test-case-list">
              <div 
                v-for="(testCase, index) in manualTestCases" 
                :key="index"
                class="test-case-item"
              >
                <span class="case-number">{{ index + 1 }}</span>
                <span class="case-cities">{{ testCase.cityList.join(' → ') }}</span>
                <el-button 
                  type="danger" 
                  size="small" 
                  text
                  @click="removeManualTestCase(index)"
                >
                  删除
                </el-button>
              </div>
            </div>
          </div>
        </div>
      </el-tab-pane>
      
      <!-- 自动生成模式 -->
      <el-tab-pane label="自动生成" name="auto">
        <div class="tab-content">
          <div class="input-section">
            <div class="input-row">
              <label>测试用例数量：</label>
              <el-input-number
                v-model="testCaseCount"
                :min="1"
                :step="1"
                style="width: 150px;"
              />
            </div>
            
            <el-button 
              type="primary" 
              @click="handleGenerateSequences"
              :disabled="testCaseCount < 1 || generatingSequences"
              :loading="generatingSequences"
              style="margin-top: 16px;"
            >
              {{ generatingSequences ? '生成中...' : '生成序列' }}
            </el-button>
          </div>
          
          <div v-if="autoTestCases.length > 0" class="test-cases-section">
            <h3>生成的测试用例（{{ autoTestCases.length }}个）：</h3>
            <div class="test-case-list">
              <div 
                v-for="(testCase, index) in autoTestCases" 
                :key="index"
                class="test-case-item"
              >
                <span class="case-number">{{ index + 1 }}</span>
                <span class="case-cities">
                  {{ testCase.startCity }} → {{ testCase.endCity }}
                </span>
              </div>
            </div>
          </div>
        </div>
      </el-tab-pane>
    </el-tabs>
    
    <!-- 测试控制区域 -->
    <div class="test-control-section">
      <div class="test-info">
        <span>总测试用例数：{{ totalTestCases }}</span>
        <span v-if="testing" class="test-progress">
          当前进度：{{ currentTestIndex }} / {{ totalTestCases }}
        </span>
      </div>
      <el-progress 
        v-if="testing && totalTestCases > 0"
        :percentage="Math.round((currentTestIndex / totalTestCases) * 100)"
        :stroke-width="8"
        style="margin-top: 12px; margin-bottom: 16px;"
      />
      
      <div class="test-buttons">
        <el-button 
          type="primary" 
          @click="handleStartTest"
          :disabled="totalTestCases === 0 || testing"
          :loading="testing"
        >
          {{ testing ? '测试中...' : '开始测试' }}
        </el-button>
        <el-button 
          @click="handleStopTest"
          :disabled="!testing"
        >
          停止测试
        </el-button>
        <el-button 
          @click="handleClearResults"
          :disabled="testing || testResults.length === 0"
        >
          清空结果
        </el-button>
        <el-button 
          type="success"
          @click="handleExportResults"
          :disabled="testResults.length === 0"
        >
          导出结果
        </el-button>
      </div>
    </div>
    
    <!-- 测试结果区域 -->
    <div v-if="testResults.length > 0" class="test-results-section">
      <h3>测试结果（{{ testResults.length }}条）：</h3>
      <div class="results-table-wrapper">
        <table class="results-table">
          <thead>
            <tr>
              <th>序号</th>
              <th>城市序列</th>
              <th>城市数</th>
              <th>序列保持度</th>
              <th>可读性</th>
              <th>角度均值(°)</th>
              <th>角度变异系数</th>
              <th>面积-权重相关性</th>
              <th>平均面积误差率</th>
              <th>平均长宽比</th>
              <th>语义信息密度</th>
              <th>运行效率(ms)</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(result, index) in testResults" :key="index">
              <td>{{ index + 1 }}</td>
              <td class="sequence-cell">{{ result.linearSequence }}</td>
              <td>{{ result.cityNodeCount }}</td>
              <td>{{ result.sequenceContinuity.toFixed(3) }}</td>
              <td>{{ result.readability.toFixed(3) }}</td>
              <td>{{ result.angleMean !== undefined ? result.angleMean.toFixed(2) : '-' }}</td>
              <td>{{ result.angleCoefficientOfVariation !== undefined ? (result.angleCoefficientOfVariation / 100).toFixed(3) : '-' }}</td>
              <td>{{ result.areaWeightCorrelation.toFixed(4) }}</td>
              <td>{{ result.averageAreaErrorRate !== undefined ? (result.averageAreaErrorRate / 100).toFixed(3) : '-' }}</td>
              <td>{{ result.averageAspectRatio.toFixed(4) }}</td>
              <td>{{ result.semanticDensity.toFixed(6) }}</td>
              <td>{{ result.renderEfficiency }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, nextTick } from 'vue';
import { ElMessage, ElMessageBox } from 'element-plus';
import { usePoiStore } from '@/stores/poiStore';
import { 
  getRouteByCityNames, 
  generateRandomCityPairs
} from '@/utils/batchTestUtils';
import AMapLoader from '@amap/amap-jsapi-loader';
import * as d3 from 'd3';
import cloud from 'd3-cloud';
import { StripLayout, SpiralLayout, PivotLayout } from '@/utils/treemapLayouts';
import { cityNameToPinyin } from '@/utils/cityNameToPinyin';
import { normalizeCityName } from '@/utils/normalizeCityName';
import { loadGeoJson } from '@/utils/geojsonLoader';
import * as turf from '@turf/turf';

const poiStore = usePoiStore();

// 标签页
const activeTab = ref('manual');

// 手动输入
const manualCityInput = ref('');
const manualTestCases = ref([]);

// 自动生成
const testCaseCount = ref(10);
const generatingSequences = ref(false);
const autoTestCases = ref([]);

// 测试状态
const testing = ref(false);
const currentTestIndex = ref(0);
const testResults = ref([]);
const shouldStop = ref(false);

// 高德地图全局对象
let amapGlobal = null;

// 计算总测试用例数
const totalTestCases = computed(() => {
  if (activeTab.value === 'manual') {
    return manualTestCases.value.length;
  } else {
    return autoTestCases.value.length;
  }
});

// 初始化高德地图
onMounted(async () => {
  try {
    amapGlobal = await AMapLoader.load({
      key: '80838eddfb922202b289fd1ad6fa4e58',
      version: '2.0',
      plugins: [
        'AMap.Driving',
        'AMap.GeometryUtil',
      ],
    });
  } catch (error) {
    console.error('高德地图加载失败:', error);
    ElMessage.error('高德地图加载失败，批量测试功能不可用');
  }
});

// 处理手动输入
const handleManualInput = () => {
  const cities = manualCityInput.value
    .split(/[,\n，]/)
    .map(c => c.trim())
    .filter(c => c.length > 0);
  
  if (cities.length === 0) {
    ElMessage.warning('请输入至少一个城市名');
    return;
  }
  
  manualTestCases.value.push({
    cityList: cities,
    startCity: cities[0],
    endCity: cities[cities.length - 1]
  });
  
  manualCityInput.value = '';
  ElMessage.success(`已添加测试用例：${cities.join(' → ')}`);
};

// 删除手动测试用例
const removeManualTestCase = (index) => {
  manualTestCases.value.splice(index, 1);
};

// 生成序列
const handleGenerateSequences = () => {
  generatingSequences.value = true;
  autoTestCases.value = [];
  
  try {
    // 只生成随机城市对，不获取路径
    const cityPairs = generateRandomCityPairs(testCaseCount.value);
    
    autoTestCases.value = cityPairs.map(pair => ({
      startCity: pair.startCity,
      endCity: pair.endCity,
      cityList: null // 将在测试时获取
    }));
    
    ElMessage.success(`成功生成 ${autoTestCases.value.length} 个测试用例`);
  } catch (error) {
    console.error('生成序列失败:', error);
    ElMessage.error('生成序列失败：' + error.message);
  } finally {
    generatingSequences.value = false;
  }
};

// 开始测试
const handleStartTest = async () => {
  if (totalTestCases.value === 0) {
    ElMessage.warning('没有可用的测试用例');
    return;
  }
  
  const confirmed = await ElMessageBox.confirm(
    `确定要开始测试 ${totalTestCases.value} 个用例吗？`,
    '确认测试',
    {
      confirmButtonText: '确定',
      cancelButtonText: '取消',
      type: 'info',
    }
  ).catch(() => false);
  
  if (!confirmed) return;
  
  testing.value = true;
  shouldStop.value = false;
  currentTestIndex.value = 0;
  testResults.value = [];
  
  const testCases = activeTab.value === 'manual' 
    ? manualTestCases.value 
    : autoTestCases.value;
  
  // 确保POI数据已加载
  await poiStore.loadDefaultData();
  
  // 确保高德地图已加载
  if (!amapGlobal) {
    ElMessage.error('高德地图未加载，无法进行测试');
    testing.value = false;
    return;
  }
  
  // 逐个执行测试
  for (let i = 0; i < testCases.length; i++) {
    if (shouldStop.value) {
      ElMessage.info('测试已停止');
      break;
    }
    
    currentTestIndex.value = i + 1;
    
    try {
      const testCase = testCases[i];
      const result = await executeSingleTest(testCase, amapGlobal);
      testResults.value.push(result);
    } catch (error) {
      console.error(`测试用例 ${i + 1} 失败:`, error);
      ElMessage.warning(`测试用例 ${i + 1} 失败：${error.message}`);
    }
  }
  
  testing.value = false;
  ElMessage.success(`测试完成！共完成 ${testResults.value.length} 个用例`);
};

// 执行单个测试用例
const executeSingleTest = async (testCase, amapGlobal) => {
  const renderStartTime = Date.now();
  
  // 获取真实的画布大小（从 TagCloudCanvas 的 wrapperRef 获取）
  let canvasWidth = 800;
  let canvasHeight = 600;
  
  // 等待 DOM 更新，确保 TagCloudCanvas 已经渲染
  await nextTick();
  await new Promise(resolve => setTimeout(resolve, 100)); // 给一点时间让 DOM 完全渲染
  
  try {
    // 优先尝试获取 TagCloudCanvas 的实际画布大小
    const canvasWrapper = document.querySelector('.tagcloud-panel .canvas-wrapper');
    if (canvasWrapper) {
      const rect = canvasWrapper.getBoundingClientRect();
      if (rect.width > 0 && rect.height > 0) {
        canvasWidth = Math.floor(rect.width);
        canvasHeight = Math.floor(rect.height);
        console.log(`[批量测试] 获取到真实画布大小: ${canvasWidth}x${canvasHeight}`);
      }
    }
    
    // 如果 wrapper 没有有效大小，尝试从 SVG 元素获取
    if (canvasWidth === 800 && canvasHeight === 600) {
      const svgElement = document.querySelector('.tagcloud-panel svg');
      if (svgElement) {
        const svgRect = svgElement.getBoundingClientRect();
        if (svgRect.width > 0 && svgRect.height > 0) {
          canvasWidth = Math.floor(svgRect.width);
          canvasHeight = Math.floor(svgRect.height);
          console.log(`[批量测试] 从SVG元素获取大小: ${canvasWidth}x${canvasHeight}`);
        } else if (svgElement.width && svgElement.height) {
          // 尝试从 SVG 的 width/height 属性获取
          canvasWidth = svgElement.width.baseVal.value || 800;
          canvasHeight = svgElement.height.baseVal.value || 600;
          console.log(`[批量测试] 从SVG属性获取大小: ${canvasWidth}x${canvasHeight}`);
        }
      }
    }
    
    // 如果还是默认值，给出警告
    if (canvasWidth === 800 && canvasHeight === 600) {
      console.warn(`[批量测试] 无法获取真实画布大小，使用默认值: ${canvasWidth}x${canvasHeight}`);
      console.warn(`[批量测试] 提示：请确保 TagCloudCanvas 已经渲染，或者手动设置画布大小`);
    }
  } catch (error) {
    console.warn(`[批量测试] 获取画布大小失败，使用默认值:`, error);
  }
  
  // 如果是自动生成的用例，需要先获取路径和城市列表
  let cityOrder = testCase.cityList;
  if (!cityOrder && testCase.startCity && testCase.endCity) {
    try {
      const routeResult = await getRouteByCityNames(
        amapGlobal,
        testCase.startCity,
        testCase.endCity
      );
      cityOrder = routeResult.cityList;
    } catch (error) {
      throw new Error(`获取路径失败：${error.message}`);
    }
  }
  
  if (!cityOrder || cityOrder.length === 0) {
    throw new Error('城市列表为空');
  }
  
  // 筛选POI数据
  const normalizedRegions = new Set();
  cityOrder.forEach(name => {
    if (name) {
      normalizedRegions.add(name);
      normalizedRegions.add(normalizeCityName(name));
    }
  });
  
  const selectedPOI = poiStore.allPOI.filter(poi => {
    const cityName = poi.city || '';
    return normalizedRegions.has(cityName) || normalizedRegions.has(normalizeCityName(cityName));
  });
  
  // 编译数据
  const data = {};
  selectedPOI.forEach(poi => {
    const { pname, name_en, city, rankInChina } = poi;
    if (!data[city]) {
      data[city] = [];
    }
    const text = poiStore.fontSettings.language === 'en' && name_en ? name_en : pname;
    data[city].push({ text: text, rankInChina });
  });
  
  // 更新字体大小
  updateFontSizesForCompiledData(data, poiStore.fontSettings);
  
  // 计算城市权重
  const cityData = cityOrder.map(city => {
    return {
      city: city,
      attractions: data[city]
        ? data[city].reduce((sum, d) => sum + (d.text.length * d.size || 10), 0)
        : 0
    };
  });
  
  // 构建层次结构
  const rootData = { values: cityData };
  const root = d3.hierarchy(rootData, d => d.values)
    .sum(d => d.attractions || 0);
  
  // 应用TreeMap布局（使用真实画布大小）
  const treemap = setupTreemapLayout(poiStore.lineType || 'Pivot', canvasWidth, canvasHeight);
  treemap(root);
  
  const leaves = root.leaves();
  
  // 构建邻接矩阵
  const graph = buildAdjacencyMatrix(leaves);
  
  // 图着色
  const colorindexs = graphColoring(graph, poiStore.colorNum);
  
  // 创建虚拟SVG用于绘制词云（不显示在页面上，仅用于计算，使用真实画布大小）
  const virtualSvg = d3.create('svg').attr('width', canvasWidth).attr('height', canvasHeight);
  
  // 计算每个城市的词云位置和大小
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
  
  // 真正绘制所有词云（这是耗时的主要部分）
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
      drawWordCloud(i, virtualSvg, data, city, centerX, centerY, colorIndex, color, cloudWidth, cloudHeight, resolve);
    });
    cloudPromises.push(promise);
  });
  
  // 等待所有词云绘制完成
  await Promise.all(cloudPromises);
  
  // 计算指标（在词云绘制完成后，使用真实画布大小）
  const metrics = calculateMetrics(
    leaves,
    cityOrder,
    cityData,
    data,
    renderStartTime,
    canvasWidth,
    canvasHeight
  );
  
  return {
    ...metrics,
    testCaseIndex: currentTestIndex.value
  };
};

// 设置treemap布局
const setupTreemapLayout = (lineType, width, height) => {
  let treemap;
  switch (lineType) {
    case 'Resquarify':
      treemap = d3.treemap().size([width, height]).round(true).tile(d3.treemapResquarify);
      break;
    case 'Binary':
      treemap = d3.treemap().size([width, height]).round(true).tile(d3.treemapBinary);
      break;
    case 'Dice':
      treemap = d3.treemap().size([width, height]).round(true).tile(d3.treemapDice);
      break;
    case 'Slice':
      treemap = d3.treemap().size([width, height]).round(true).tile(d3.treemapSlice);
      break;
    case 'Strip':
      treemap = d3.treemap().size([width, height]).round(true).tile(StripLayout);
      break;
    case 'Spiral':
      treemap = d3.treemap().size([width, height]).round(true).tile(SpiralLayout);
      break;
    case 'Pivot':
      treemap = d3.treemap().size([width, height]).round(true).tile(PivotLayout);
      break;
    default:
      treemap = d3.treemap().size([width, height]).round(true).tile(d3.treemapResquarify);
  }
  return treemap;
};

// 构建邻接矩阵
const buildAdjacencyMatrix = (nodes) => {
  const AdjacencyMatrix = [];
  nodes.forEach(a => {
    const temp = [];
    nodes.forEach(b => {
      if (a == b) {
        temp.push(0);
      } else {
        temp.push(areNodesAdjacent(a, b) ? 1 : 0);
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
    (((aBottom >= bTop && aTop <= bTop) || (bTop <= aTop && aTop <= aBottom)) && (bRight === aLeft));
  return adjacentVertically || adjacentHorizontally;
};

// 图着色
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
    if (vertex === graph.length) return true;
    for (let color = 1; color <= m; color++) {
      if (isSafe(vertex, color)) {
        colorindexs[vertex] = color;
        if (graphColoringUtil(vertex + 1)) return true;
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

// 绘制单个词云（从TagCloudCanvas.vue复制）
const drawWordCloud = (i, svg, cities, city, x, y, colorIndex, color, width, height, resolve) => {
  if (!isFinite(x) || !isFinite(y)) { 
    if (resolve) resolve(); 
    return; 
  }
  const words = cities[city] ? cities[city].map(d => ({ text: d.text, size: d.size, color: color })) : [];
  if (words.length == 0) { 
    if (resolve) resolve(); 
    return; 
  }
  // 根据语言设置选择城市名
  let cityName = poiStore.fontSettings.language === 'en' 
    ? cityNameToPinyin(city) 
    : city;
  // 如果需要显示序号，在城市名前添加序号
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

// 更新字体大小
const updateFontSizesForCompiledData = (compiledData, fontSettings) => {
  const allPoiArr = [];
  Object.values(compiledData).forEach(arr => allPoiArr.push(...arr));
  const rankList = allPoiArr.map(p => parseInt(p.rankInChina)).filter(r => r > 0 && isFinite(r));
  if (!rankList.length) { 
    allPoiArr.forEach(p => p.size = fontSettings.minFontSize); 
    return; 
  }
  const minRank = Math.min(...rankList);
  const maxRank = Math.max(...rankList);
  if (!isFinite(minRank) || !isFinite(maxRank) || minRank <= 0 || maxRank <= 0) { 
    allPoiArr.forEach(p => p.size = fontSettings.minFontSize); 
    return; 
  }
  const logMin = Math.log(minRank);
  const logMax = Math.log(maxRank);
  allPoiArr.forEach(poi => {
    const r = parseInt(poi.rankInChina);
    if (!r || r <= 0 || !isFinite(r)) { 
      poi.size = fontSettings.minFontSize; 
      return; 
    }
    const logR = Math.log(r);
    if (logMax - logMin === 0) { 
      poi.size = fontSettings.minFontSize; 
      return; 
    }
    let size = fontSettings.minFontSize + (fontSettings.maxFontSize - fontSettings.minFontSize) * (logMax - logR) / (logMax - logMin);
    poi.size = (isFinite(size) && size > 0) ? size : fontSettings.minFontSize;
  });
};

// 计算指标（从TagCloudCanvas.vue复制）
const calculateMetrics = (leaves, cityOrder, cityData, data, renderStartTime, canvasWidth, canvasHeight) => {
  const linearSequence = cityOrder.join('-');
  const cityNodeCount = cityOrder.length;
  
  const cityToLeaf = new Map();
  leaves.forEach(leaf => {
    cityToLeaf.set(leaf.data.city, leaf);
  });
  
  // 序列保持度
  let adjacentPairs = 0;
  let stillAdjacentPairs = 0;
  for (let i = 0; i < cityOrder.length - 1; i++) {
    const city1 = cityOrder[i];
    const city2 = cityOrder[i + 1];
    adjacentPairs++;
    const leaf1 = cityToLeaf.get(city1);
    const leaf2 = cityToLeaf.get(city2);
    if (leaf1 && leaf2 && areNodesAdjacent(leaf1, leaf2)) {
      stillAdjacentPairs++;
    }
  }
  const sequenceContinuity = adjacentPairs > 0 ? stillAdjacentPairs / adjacentPairs : 0;
  
  // 可读性
  let noOffsetCount = 0;
  let totalVectorPairs = 0;
  const directionVectors = [];
  for (let i = 0; i < cityOrder.length - 1; i++) {
    const city1 = cityOrder[i];
    const city2 = cityOrder[i + 1];
    const leaf1 = cityToLeaf.get(city1);
    const leaf2 = cityToLeaf.get(city2);
    if (leaf1 && leaf2) {
      const centerX1 = (leaf1.x0 + leaf1.x1) / 2;
      const centerY1 = (leaf1.y0 + leaf1.y1) / 2;
      const centerX2 = (leaf2.x0 + leaf2.x1) / 2;
      const centerY2 = (leaf2.y0 + leaf2.y1) / 2;
      const dx = centerX2 - centerX1;
      const dy = centerY2 - centerY1;
      directionVectors.push({
        fromCity: city1,
        toCity: city2,
        dx: dx,
        dy: dy,
        angle: Math.atan2(dy, dx)
      });
    }
  }
  const angleDiffs = []; // 存储所有角度差值，用于计算均值和方差
  for (let i = 0; i < directionVectors.length - 1; i++) {
    const vector1 = directionVectors[i];
    const vector2 = directionVectors[i + 1];
    totalVectorPairs++;
    const angle1 = vector1.angle;
    const angle2 = vector2.angle;
    let angleDiff = Math.abs(angle2 - angle1) * 180 / Math.PI;
    if (angleDiff > 180) {
      angleDiff = 360 - angleDiff;
    }
    angleDiffs.push(angleDiff); // 保存角度差值
    if (angleDiff <= 6) {
      noOffsetCount++;
    }
  }
  const readability = totalVectorPairs > 0 ? noOffsetCount / totalVectorPairs : 0;
  
  // 计算角度均值和变异系数
  let angleMean = 0;
  let angleCoefficientOfVariation = 0;
  if (angleDiffs.length > 0) {
    // 计算均值
    angleMean = angleDiffs.reduce((sum, angle) => sum + angle, 0) / angleDiffs.length;
    // 计算变异系数（标准差/均值 × 100%，与 TagCloudCanvas.vue 保持一致）
    if (angleDiffs.length > 1 && angleMean > 0) {
      const sumSquaredDiff = angleDiffs.reduce((sum, angle) => {
        const diff = angle - angleMean;
        return sum + diff * diff;
      }, 0);
      const variance = sumSquaredDiff / angleDiffs.length;
      const standardDeviation = Math.sqrt(variance);
      angleCoefficientOfVariation = (standardDeviation / angleMean) * 100; // 变异系数（百分比形式，与手动测试保持一致）
    }
  }
  
  // 面积-权重相关性
  const weights = [];
  const areas = [];
  cityOrder.forEach(city => {
    const cityDataItem = cityData.find(d => d.city === city);
    const leaf = cityToLeaf.get(city);
    if (cityDataItem && leaf) {
      weights.push(cityDataItem.attractions || 0);
      const area = (leaf.x1 - leaf.x0) * (leaf.y1 - leaf.y0);
      areas.push(area);
    }
  });
  let areaWeightCorrelation = 0;
  if (weights.length > 1 && areas.length > 1) {
    const meanWeight = weights.reduce((a, b) => a + b, 0) / weights.length;
    const meanArea = areas.reduce((a, b) => a + b, 0) / areas.length;
    let numerator = 0;
    let sumSqWeight = 0;
    let sumSqArea = 0;
    for (let i = 0; i < weights.length; i++) {
      const weightDiff = weights[i] - meanWeight;
      const areaDiff = areas[i] - meanArea;
      numerator += weightDiff * areaDiff;
      sumSqWeight += weightDiff * weightDiff;
      sumSqArea += areaDiff * areaDiff;
    }
    const denominator = Math.sqrt(sumSqWeight * sumSqArea);
    areaWeightCorrelation = denominator > 0 ? numerator / denominator : 0;
  }
  
  // 平均面积误差率：理论应分配的面积与实际分配的面积之间的误差率
  const totalWeight = cityData.reduce((sum, d) => sum + (d.attractions || 0), 0);
  const totalCanvasArea = canvasWidth * canvasHeight;
  const areaErrorRates = [];
  
  cityOrder.forEach(city => {
    const cityDataItem = cityData.find(d => d.city === city);
    const leaf = cityToLeaf.get(city);
    
    if (cityDataItem && leaf) {
      const weight = cityDataItem.attractions || 0;
      const expectedArea = totalWeight > 0 ? (weight / totalWeight) * totalCanvasArea : 0;
      const actualArea = (leaf.x1 - leaf.x0) * (leaf.y1 - leaf.y0);
      const error = Math.abs(actualArea - expectedArea);
      const errorRate = expectedArea > 0 ? (error / expectedArea) * 100 : 0;
      
      areaErrorRates.push({
        city: city,
        expectedArea: expectedArea.toFixed(2),
        actualArea: actualArea.toFixed(2),
        errorRate: errorRate.toFixed(2)
      });
    }
  });
  
  const averageAreaErrorRate = areaErrorRates.length > 0 
    ? areaErrorRates.reduce((sum, item) => sum + parseFloat(item.errorRate), 0) / areaErrorRates.length 
    : 0;
  
  // 平均长宽比
  let totalAAR = 0;
  let validLeaves = 0;
  leaves.forEach(leaf => {
    const width = leaf.x1 - leaf.x0;
    const height = leaf.y1 - leaf.y0;
    if (width > 0 && height > 0) {
      const aar = width > height ? width / height : height / width;
      totalAAR += aar;
      validLeaves++;
    }
  });
  const averageAspectRatio = validLeaves > 0 ? totalAAR / validLeaves : 0;
  
  // 语义信息密度
  let totalTags = 0;
  cityOrder.forEach(city => {
    const cityTags = data[city] ? data[city].length : 0;
    totalTags += cityTags;
  });
  const semanticDensity = totalCanvasArea > 0 ? totalTags / totalCanvasArea : 0;
  
  // 运行效率
  const renderEndTime = Date.now();
  const renderEfficiency = renderEndTime - renderStartTime;
  
  return {
    linearSequence,
    cityNodeCount,
    sequenceContinuity,
    readability,
    angleMean,
    angleCoefficientOfVariation,
    areaWeightCorrelation,
    averageAreaErrorRate,
    areaErrorRates,
    averageAspectRatio,
    semanticDensity,
    renderEfficiency
  };
};

// 停止测试
const handleStopTest = () => {
  shouldStop.value = true;
  ElMessage.info('正在停止测试...');
};

// 清空结果
const handleClearResults = () => {
  testResults.value = [];
  ElMessage.success('已清空测试结果');
};

// 导出结果
const handleExportResults = () => {
  if (testResults.value.length === 0) {
    ElMessage.warning('没有可导出的结果');
    return;
  }
  
  // 转换为CSV格式
  const headers = [
    '序号',
    '城市序列',
    '城市节点数量',
    '序列保持度',
    '可读性',
    '角度均值(°)',
    '角度变异系数',
    '面积-权重相关性',
    '平均面积误差率',
    '平均长宽比AAR',
    '语义信息密度',
    '运行效率(ms)'
  ];
  
  const rows = testResults.value.map((result, index) => [
    index + 1,
    result.linearSequence,
    result.cityNodeCount,
    result.sequenceContinuity.toFixed(3),
    result.readability.toFixed(3),
    result.angleMean !== undefined ? result.angleMean.toFixed(2) : '-',
    result.angleCoefficientOfVariation !== undefined ? (result.angleCoefficientOfVariation / 100).toFixed(3) : '-',
    result.areaWeightCorrelation.toFixed(4),
    result.averageAreaErrorRate !== undefined ? (result.averageAreaErrorRate / 100).toFixed(3) : '-',
    result.averageAspectRatio.toFixed(4),
    result.semanticDensity.toFixed(6),
    result.renderEfficiency
  ]);
  
  const csvContent = [
    headers.join(','),
    ...rows.map(row => row.map(cell => `"${cell}"`).join(','))
  ].join('\n');
  
  // 添加BOM以支持中文
  const BOM = '\uFEFF';
  const blob = new Blob([BOM + csvContent], { type: 'text/csv;charset=utf-8;' });
  const link = document.createElement('a');
  const url = URL.createObjectURL(blob);
  link.setAttribute('href', url);
  link.setAttribute('download', `批量测试结果_${new Date().toISOString().slice(0, 10)}.csv`);
  link.style.visibility = 'hidden';
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
  
  ElMessage.success('结果已导出');
};
</script>

<style scoped>
.batch-test-panel {
  padding: 24px;
  height: 100%;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.panel-header {
  margin-bottom: 20px;
}

.panel-header h2 {
  margin: 0;
  font-size: 20px;
  font-weight: 600;
  color: #1f2333;
}

.test-tabs {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.tab-content {
  flex: 1;
  overflow-y: auto;
  padding: 16px 0;
}

.input-section {
  margin-bottom: 24px;
}

.input-section label {
  display: block;
  margin-bottom: 8px;
  font-size: 14px;
  color: #606266;
  font-weight: 500;
}

.city-input {
  width: 100%;
}

.input-row {
  display: flex;
  align-items: center;
  gap: 12px;
}

.input-row label {
  margin-bottom: 0;
  min-width: 120px;
}

.filter-inputs {
  display: flex;
  align-items: center;
  gap: 8px;
}

.test-cases-section {
  margin-top: 24px;
}

.test-cases-section h3 {
  margin: 0 0 12px 0;
  font-size: 16px;
  font-weight: 600;
  color: #1f2333;
}

.test-case-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
  max-height: 180px;
  overflow-y: auto;
}

.test-case-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px;
  background: #f5f7fa;
  border-radius: 8px;
  font-size: 14px;
}

.case-number {
  font-weight: 600;
  color: #409eff;
  min-width: 30px;
}

.case-cities {
  flex: 1;
  color: #606266;
}

.city-count {
  color: #909399;
  font-size: 12px;
}

.case-cities-detail {
  font-size: 12px;
  color: #909399;
  margin-left: 8px;
}

.test-control-section {
  padding: 16px 0;
  border-top: 1px solid #e4e7ed;
  margin-top: 16px;
}

.test-info {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
  font-size: 14px;
  color: #606266;
}

.test-progress {
  color: #409eff;
  font-weight: 600;
}

.test-buttons {
  display: flex;
  gap: 12px;
}

.test-results-section {
  margin-top: 24px;
  padding-top: 24px;
  border-top: 1px solid #e4e7ed;
}

.test-results-section h3 {
  margin: 0 0 16px 0;
  font-size: 16px;
  font-weight: 600;
  color: #1f2333;
}

.results-table-wrapper {
  max-height: 220px;
  overflow: auto;
  border: 1px solid #e4e7ed;
  border-radius: 8px;
}

.results-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 13px;
}

.results-table thead {
  background: #f5f7fa;
  position: sticky;
  top: 0;
  z-index: 10;
}

.results-table th {
  padding: 12px;
  text-align: left;
  font-weight: 600;
  color: #606266;
  border-bottom: 1px solid #e4e7ed;
  white-space: nowrap;
}

.results-table td {
  padding: 12px;
  border-bottom: 1px solid #e4e7ed;
  color: #606266;
}

.results-table tbody tr:hover {
  background: #f5f7fa;
}

.sequence-cell {
  max-width: 300px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
</style>

