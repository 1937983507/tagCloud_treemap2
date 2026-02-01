<template>
  <div class="app-shell">
    <HeaderBar
      ref="headerRef"
      :show-tutorial-icon="!showHelpPage && !showFeedbackPage"
      @navigate="handleNavigate"
      @start-tutorial="restartIntro"
    />
    <FeedbackPage
      v-if="showFeedbackPage && !showHelpPage"
      @navigate="handleNavigate"
    />
    <template v-else-if="!showHelpPage">
      <div class="app-body">
        <SideMenu
          ref="sideMenuRef"
          :active-panel="activePanel"
          @change-panel="handleChangePanel"
          @navigate="handleNavigate"
          @module-visibility-change="handleModuleVisibilityChange"
        />
        <div class="workspace">
          <PoiContent ref="poiContentRef" v-show="activePanel === 'content' && moduleVisibility.content" />
          <TypefacePanel v-show="activePanel === 'typeface' && moduleVisibility.typeface" />
          <ColorPanel v-show="activePanel === 'color' && moduleVisibility.color" />
          <LayoutPanel v-show="activePanel === 'layout' && moduleVisibility.layout" />
          <LinePanel v-show="activePanel === 'line' && moduleVisibility.line" />
          <BatchTestPanel 
            v-if="moduleVisibility.batchtest" 
            v-show="activePanel === 'batchtest'" 
          />
        </div>
        <SplitterBar />
        <TagCloudCanvas ref="tagCloudCanvasRef" />
      </div>
      <FooterBar @navigate="handleNavigate" />
    </template>
    <HelpPage v-else @navigate="handleNavigate" />
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue';
import introJs from 'intro.js';
import 'intro.js/minified/introjs.min.css';
import HeaderBar from '@/components/layout/HeaderBar.vue';
import FooterBar from '@/components/layout/FooterBar.vue';
import SideMenu from '@/components/layout/SideMenu.vue';
import PoiContent from '@/components/content/PoiContent.vue';
import TypefacePanel from '@/components/typeface/TypefacePanel.vue';
import ColorPanel from '@/components/color/ColorPanel.vue';
import TagCloudCanvas from '@/components/tagcloud/TagCloudCanvas.vue';
import SplitterBar from '@/components/common/SplitterBar.vue';
import LayoutPanel from '@/components/layout/LayoutPanel.vue';
import LinePanel from '@/components/line/LinePanel.vue';
import BatchTestPanel from '@/components/batchtest/BatchTestPanel.vue';
import FeedbackPage from '@/components/feedback/FeedbackPage.vue';
import HelpPage from '@/components/help/HelpPage.vue';
import { recordPageVisit } from '@/utils/statistics';

const activePanel = ref('content');
const headerRef = ref(null);
const poiContentRef = ref(null);
const tagCloudCanvasRef = ref(null);
const sideMenuRef = ref(null);
const showHelpPage = ref(false);
const showFeedbackPage = ref(false);

// 模块可见性设置
const MODULE_VISIBILITY_KEY = 'tagCloud_treemap2_moduleVisibility';
const defaultModuleVisibility = {
  content: true,
  typeface: true,
  color: true,
  layout: true,
  line: true,
  batchtest: false,
};

// 从 localStorage 读取模块可见性设置
const loadModuleVisibility = () => {
  try {
    const saved = localStorage.getItem(MODULE_VISIBILITY_KEY);
    if (saved) {
      const parsed = JSON.parse(saved);
      // 合并默认设置和保存的设置
      // 对于批量测试，如果用户没有明确设置过，使用默认值 false
      const result = { ...defaultModuleVisibility, ...parsed };
      // 如果保存的设置中没有 batchtest 字段，使用默认值 false
      if (!('batchtest' in parsed)) {
        result.batchtest = defaultModuleVisibility.batchtest;
      }
      return result;
    }
  } catch (error) {
    console.warn('读取模块可见性设置失败:', error);
  }
  return { ...defaultModuleVisibility };
};

const moduleVisibility = ref(loadModuleVisibility());

let firstIntroStarted = false;
let currentIntro = null;

// localStorage key for tutorial preference
const TUTORIAL_DISABLED_KEY = 'tagCloud_treemap2_tutorialDisabled';

// Check if tutorial should be disabled
const shouldDisableTutorial = () => {
  return localStorage.getItem(TUTORIAL_DISABLED_KEY) === 'true';
};

// Save tutorial preference
const saveTutorialPreference = (disabled) => {
  localStorage.setItem(TUTORIAL_DISABLED_KEY, disabled ? 'true' : 'false');
};

// Get current tutorial preference
const getTutorialPreference = () => {
  return localStorage.getItem(TUTORIAL_DISABLED_KEY) === 'true';
};

// Expose function to window for inline event handler
if (typeof window !== 'undefined') {
  window.__saveTutorialPreference_treemap2 = saveTutorialPreference;
  window.__getTutorialPreference_treemap2 = getTutorialPreference;
}

// Helper function to add checkbox to intro content
const addCheckboxToIntro = (content) => {
  // Always read from localStorage to get the latest value
  const isChecked = getTutorialPreference();
  const checkedAttr = isChecked ? 'checked' : '';
  // Inline onchange: immediately save to localStorage and sync all checkboxes
  const checkboxHtml = `<div style="margin-top:16px;padding-top:16px;border-top:1px solid #e2e8f0;text-align:left;"><label style="display:flex;align-items:center;cursor:pointer;font-size:13px;color:#64748b;"><input type="checkbox" class="tutorial-disable-checkbox-treemap2" ${checkedAttr} style="margin-right:8px;cursor:pointer;width:16px;height:16px;" onchange="if(window.__saveTutorialPreference_treemap2) { window.__saveTutorialPreference_treemap2(this.checked); const allCb = document.querySelectorAll('.tutorial-disable-checkbox-treemap2'); allCb.forEach(cb => cb.checked = this.checked); }" /><span>最近不再默认显示此引导</span></label></div>`;
  return content + checkboxHtml;
};

const handleChangePanel = (panel) => {
  // 检查该面板是否可见
  if (moduleVisibility.value[panel] === false) {
    // 如果不可见，切换到第一个可见的面板
    const visiblePanels = Object.keys(moduleVisibility.value).filter(
      key => moduleVisibility.value[key] === true
    );
    if (visiblePanels.length > 0) {
      activePanel.value = visiblePanels[0];
    }
    return;
  }
  activePanel.value = panel;
};

// 处理模块可见性变化
const handleModuleVisibilityChange = ({ moduleKey, visible }) => {
  moduleVisibility.value[moduleKey] = visible;
  // 如果当前激活的面板被隐藏，切换到第一个可见的面板
  if (!visible && activePanel.value === moduleKey) {
    const visiblePanels = Object.keys(moduleVisibility.value).filter(
      key => moduleVisibility.value[key] === true && key !== moduleKey
    );
    if (visiblePanels.length > 0) {
      activePanel.value = visiblePanels[0];
    }
  }
};

const handleNavigate = (route) => {
  if (route === 'help') {
    showHelpPage.value = true;
    showFeedbackPage.value = false;
  } else if (route === 'feedback') {
    showFeedbackPage.value = true;
    showHelpPage.value = false;
  } else if (route === 'home') {
    showHelpPage.value = false;
    showFeedbackPage.value = false;
  } else {
    console.log('navigate to', route);
  }
};

const getHeaderElement = () => {
  if (headerRef.value?.$el) return headerRef.value.$el;
  return document.querySelector('header.header');
};

const getSideMenuElement = () => {
  return document.querySelector('.side-menu');
};

const getMapElement = () => {
  if (poiContentRef.value?.$el) {
    const mapWrapper = poiContentRef.value.$el.querySelector('.map-wrapper');
    if (mapWrapper) return mapWrapper;
  }
  return document.querySelector('.map-wrapper');
};

const getGraphElement = () => {
  if (poiContentRef.value?.$el) {
    const graphEl =
      poiContentRef.value.$el.querySelector('.graph-panel') ||
      poiContentRef.value.$el.querySelector('.table-card-placeholder');
    if (graphEl) return graphEl;
  }
  return document.querySelector('.graph-panel') || document.querySelector('.table-card-placeholder');
};

const getTagCloudPanelElement = () => {
  if (tagCloudCanvasRef.value?.$el) {
    const headEl = tagCloudCanvasRef.value.$el.querySelector('.panel-head');
    if (headEl) return headEl;
  }
  return document.querySelector('.tagcloud-panel .panel-head');
};

const getCanvasElement = () => {
  if (tagCloudCanvasRef.value?.$el) {
    const wrapperEl = tagCloudCanvasRef.value.$el.querySelector('.canvas-wrapper');
    if (wrapperEl) return wrapperEl;
  }
  return document.querySelector('.tagcloud-panel .canvas-wrapper') || document.querySelector('.tagcloud-panel svg');
};

const getTutorialButtonElement = () => {
  const tutorialBtn = document.querySelector('[data-intro-tutorial="tutorial-btn"]');
  if (tutorialBtn) return tutorialBtn;
  if (headerRef.value?.$el) {
    return headerRef.value.$el.querySelector('.tutorial-icon-link') || getHeaderElement();
  }
  return getHeaderElement();
};

const createIntro = () => {
  // Check if introJs is available
  if (!introJs || typeof introJs.tour !== 'function') {
    console.error('Intro.js is not properly loaded');
    throw new Error('Intro.js is not properly loaded');
  }
  
  const intro = introJs.tour();
  
  // Build steps array with checkbox in each step
  const steps = [
    {
      intro: addCheckboxToIntro(
        '<div style="text-align:center;padding:8px 0;"><div style="font-size:18px;font-weight:600;margin-bottom:8px;">欢迎体验地名标签云</div><div style="font-size:13px;color:#64748b;">我们将带您快速认识核心功能，帮助更顺利地生成路线标签云。</div></div>'
      ),
    },
    {
      element: getHeaderElement(),
      intro: addCheckboxToIntro(
        '<div style="line-height:1.6;"><strong style="font-size:16px;color:#1f2333;">导航栏</strong><br/><span style="color:#64748b;">这里可以返回首页、查看帮助或反馈意见。点击右上角的<span style="color:#399ceb;">"引导教程"</span>图标可再次查看本引导。</span></div>'
      ),
    },
    {
      element: getSideMenuElement(),
      intro: addCheckboxToIntro(
        '<div style="line-height:1.6;"><strong style="font-size:16px;color:#1f2333;">侧边面板</strong><br/><span style="color:#64748b;">按照"内容 → 字体 → 配色 → 布局 → 线条"的顺序逐步完善展示效果。</span></div>'
      ),
    },
    {
      element: getMapElement(),
      intro: addCheckboxToIntro(
        '<div style="line-height:1.6;"><strong style="font-size:16px;color:#1f2333;">地图绘制区</strong><br/><span style="color:#64748b;">在此绘制折线或选择路线，系统将根据经过的城市提取景点数据。</span></div>'
      ),
    },
    {
      element: getGraphElement(),
      intro: addCheckboxToIntro(
        '<div style="line-height:1.6;"><strong style="font-size:16px;color:#1f2333;">数据分析视图</strong><br/><span style="color:#64748b;">实时查看城市关联、路径统计等信息，帮助评估线路覆盖情况。</span></div>'
      ),
    },
    {
      element: getTagCloudPanelElement(),
      intro: addCheckboxToIntro(
        '<div style="line-height:1.6;"><strong style="font-size:16px;color:#1f2333;">标签云控制面板</strong><br/><span style="color:#64748b;">点击"运行生成标签云"并根据需求调整导出、字号、配色等参数。</span></div>'
      ),
    },
    {
      element: getCanvasElement(),
      intro: addCheckboxToIntro(
        '<div style="line-height:1.6;"><strong style="font-size:16px;color:#1f2333;">标签云画布</strong><br/><span style="color:#64748b;">生成结果会显示在此处，可搭配缩放、导出等操作完成展示。</span></div>'
      ),
    },
    {
      element: getTutorialButtonElement(),
      intro: addCheckboxToIntro(
        '<div style="text-align:center;line-height:1.6;"><div style="font-size:20px;margin-bottom:12px;">🎉 引导完成</div><div style="color:#64748b;margin-bottom:12px;">随时点击右上角的<span style="color:#399ceb;">"引导教程"</span>图标重新查看操作提示。</div><div style="font-size:12px;color:#94a3b8;">祝您创作顺利！</div></div>'
      ),
    },
  ];
  
  intro.addSteps(steps);

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
    scrollToElement: true,
    scrollPadding: 20,
    overlayOpacity: 0.4,
    tooltipPosition: 'auto',
    exitOnOverlayClick: true,
    exitOnEsc: true,
    keyboardNavigation: true,
    tooltipRenderAsHtml: true,
  });

  // Helper function to sync all checkboxes from localStorage
  const syncAllCheckboxes = () => {
    // Always read latest value from localStorage
    const isDisabled = getTutorialPreference();
    const checkboxes = document.querySelectorAll('.tutorial-disable-checkbox-treemap2');
    checkboxes.forEach((checkbox) => {
      // Update checkbox state from localStorage
      checkbox.checked = isDisabled;
      // Add event listener if not already attached
      if (!checkbox.hasAttribute('data-listener-attached')) {
        checkbox.setAttribute('data-listener-attached', 'true');
        checkbox.addEventListener('change', (e) => {
          // Immediately save to localStorage when user clicks
          saveTutorialPreference(e.target.checked);
          // Immediately sync all checkboxes
          const allCheckboxes = document.querySelectorAll('.tutorial-disable-checkbox-treemap2');
          allCheckboxes.forEach((cb) => {
            cb.checked = e.target.checked;
          });
        });
      }
    });
  };
  
  // Sync checkboxes when step changes - always read from localStorage
  // Use multiple attempts to ensure DOM is fully rendered
  if (typeof intro.onchange === 'function') {
    intro.onchange(() => {
      // Use requestAnimationFrame to ensure browser has rendered
      requestAnimationFrame(() => {
        nextTick(() => {
          syncAllCheckboxes();
          // Also try after a short delay to catch any late rendering
          setTimeout(() => {
            syncAllCheckboxes();
          }, 100);
        });
      });
    });
  }
  
  // Also sync checkboxes when intro starts (for the first step)
  // Use onstart if available, otherwise sync will happen on first step change
  if (typeof intro.onstart === 'function') {
    intro.onstart(() => {
      nextTick(() => {
        syncAllCheckboxes();
        setTimeout(() => {
          syncAllCheckboxes();
        }, 50);
      });
    });
  }
  
  // Use MutationObserver to catch tooltip rendering and sync checkboxes
  let syncTimeout = null;
  const observer = new MutationObserver((mutations) => {
    // Debounce to avoid too frequent updates
    if (syncTimeout) {
      clearTimeout(syncTimeout);
    }
    syncTimeout = setTimeout(() => {
      // Check if tooltip exists and has checkbox
      const tooltip = document.querySelector('.introjs-tooltip');
      if (tooltip) {
        const checkbox = tooltip.querySelector('.tutorial-disable-checkbox-treemap2');
        if (checkbox) {
          // Sync from localStorage
          const isDisabled = getTutorialPreference();
          checkbox.checked = isDisabled;
          // Ensure listener is attached
          if (!checkbox.hasAttribute('data-listener-attached')) {
            checkbox.setAttribute('data-listener-attached', 'true');
            checkbox.addEventListener('change', (e) => {
              saveTutorialPreference(e.target.checked);
              const allCheckboxes = document.querySelectorAll('.tutorial-disable-checkbox-treemap2');
              allCheckboxes.forEach((cb) => {
                cb.checked = e.target.checked;
              });
            });
          }
        }
      }
    }, 10);
  });
  
  // Start observing when intro starts
  intro.onComplete(() => {
    observer.disconnect();
    if (syncTimeout) {
      clearTimeout(syncTimeout);
    }
  });
  
  intro.onExit(() => {
    observer.disconnect();
    if (syncTimeout) {
      clearTimeout(syncTimeout);
    }
  });
  
  // Observe introjs tooltip container for changes
  const observeTooltip = () => {
    const tooltipContainer = document.querySelector('.introjs-tooltipReferenceLayer') || document.body;
    observer.observe(tooltipContainer, {
      childList: true,
      subtree: true,
      attributes: false,
    });
  };
  
  // Start observing after a short delay to ensure intro is initialized
  setTimeout(() => {
    observeTooltip();
  }, 100);

  intro.onComplete(() => {
    // Check checkbox state when completing (check any checkbox, they should all be in sync)
    const checkbox = document.querySelector('.tutorial-disable-checkbox-treemap2');
    if (checkbox) {
      saveTutorialPreference(checkbox.checked);
    }
    firstIntroStarted = false;
    currentIntro = null;
  });

  intro.onExit(() => {
    // Check checkbox state when exiting (check any checkbox, they should all be in sync)
    const checkbox = document.querySelector('.tutorial-disable-checkbox-treemap2');
    if (checkbox) {
      saveTutorialPreference(checkbox.checked);
    }
    firstIntroStarted = false;
    currentIntro = null;
  });

  return intro;
};

const restartIntro = () => {
  // Exit current intro if exists
  if (currentIntro) {
    try {
      if (typeof currentIntro.exit === 'function') {
        currentIntro.exit(true);
      } else if (typeof currentIntro.exitIntro === 'function') {
        currentIntro.exitIntro(true);
      }
    } catch (e) {
      console.warn('Error exiting current intro:', e);
    }
  }
  
  // Also try to exit any existing intro.js instance
  try {
    if (introJs && typeof introJs.exit === 'function') {
      introJs.exit(true);
    }
  } catch (e) {
    // Ignore errors
  }
  
  firstIntroStarted = false;
  currentIntro = null;
  
  // Wait a bit for cleanup, then start new intro
  setTimeout(() => {
    nextTick(() => {
      try {
        const intro = createIntro();
        currentIntro = intro;
        // Ensure intro.js is available
        if (!intro || typeof intro.start !== 'function') {
          console.error('Intro.js not properly initialized');
          return;
        }
        console.log('Starting intro.js tour...');
        intro.start();
      } catch (error) {
        console.error('Error starting intro:', error);
        firstIntroStarted = false;
        currentIntro = null;
      }
    });
  }, 200);
};

const initIntro = () => {
  // Check if user has disabled tutorial
  if (shouldDisableTutorial()) {
    return;
  }
  if (firstIntroStarted || showHelpPage.value || showFeedbackPage.value) return;
  firstIntroStarted = true;
  nextTick(() => {
    try {
      const intro = createIntro();
      currentIntro = intro;
      // Ensure intro.js is available
      if (!intro || typeof intro.start !== 'function') {
        console.error('Intro.js not properly initialized');
        firstIntroStarted = false;
        currentIntro = null;
        return;
      }
      intro.start();
    } catch (error) {
      console.error('Error starting intro:', error);
      firstIntroStarted = false;
      currentIntro = null;
    }
  });
};

onMounted(async () => {
  // 记录页面访问，等待完成后触发事件通知 FooterBar 更新
  try {
    await recordPageVisit();
    // 触发自定义事件，通知 FooterBar 访问已记录，可以更新统计数据
    window.dispatchEvent(new CustomEvent('page-visit-recorded'));
  } catch (error) {
    console.warn('记录页面访问失败:', error);
    // 即使失败也触发事件，让 FooterBar 可以加载现有数据
    window.dispatchEvent(new CustomEvent('page-visit-recorded'));
  }
  
  setTimeout(() => {
    if (!showHelpPage.value && !showFeedbackPage.value) {
      initIntro();
    }
  }, 500);
});
</script>

<style scoped>
.app-shell {
  display: flex;
  flex-direction: column;
  height: 100vh;
  overflow: hidden;
  position: relative;
}

.app-body {
  flex: 1 1 auto;
  min-height: 0;
  overflow: hidden;
  display: grid;
  grid-template-columns: 108px 1fr 12px 68vw;
  background: linear-gradient(180deg, #ffffff 0%, #f7f9fc 100%);
}

.workspace {
  padding: 18px;
  background: #ffffff;
  min-height: 0;
  height: 100%;
}

.workspace > * {
  display: block;
  height: 100%;
}
</style>

