<template>
  <div
    class="lyric-window"
    :class="[lyricSetting.theme, { lyric_lock: lyricSetting.isLock }]"
    @mouseenter="handleMouseEnter"
    @mouseleave="handleMouseLeave"
  >
    <!-- 顶部控制栏 -->
    <div class="control-bar" :class="{ 'control-bar-show': showControls }">
      <div class="font-size-controls">
        <n-button-group>
          <n-button quaternary size="small" :disabled="fontSize <= 12" @click="decreaseFontSize">
            <i class="ri-subtract-line"></i>
          </n-button>
          <n-button quaternary size="small" :disabled="fontSize >= 48" @click="increaseFontSize">
            <i class="ri-add-line"></i>
          </n-button>
        </n-button-group>
      </div>
      <div class="control-buttons">
        <div class="control-button" @click="checkTheme">
          <i v-if="lyricSetting.theme === 'light'" class="ri-sun-line"></i>
          <i v-else class="ri-moon-line"></i>
        </div>
        <div class="control-button" @click="handleTop">
          <i class="ri-pushpin-line" :class="{ active: lyricSetting.isTop }"></i>
        </div>
        <div class="control-button" @click="handleLock">
          <i v-if="lyricSetting.isLock" class="ri-lock-line"></i>
          <i v-else class="ri-lock-unlock-line"></i>
        </div>
        <div class="control-button" @click="handleClose">
          <i class="ri-close-line"></i>
        </div>
      </div>
    </div>

    <!-- 歌词显示区域 -->
    <div ref="containerRef" class="lyric-container">
      <div class="lyric-scroll">
        <div class="lyric-wrapper" :style="wrapperStyle">
          <template v-if="staticData.lrcArray?.length > 0">
            <div
              v-for="(line, index) in staticData.lrcArray"
              :key="index"
              class="lyric-line"
              :style="lyricLineStyle"
              :class="{
                'lyric-line-current': index === currentIndex,
                'lyric-line-passed': index < currentIndex,
                'lyric-line-next': index === currentIndex + 1,
              }"
            >
              <div class="lyric-text" :style="{ fontSize: `${fontSize}px` }">
                <span class="lyric-text-inner" :style="getLyricStyle(index)">
                  {{ line.text || '' }}
                </span>
              </div>
              <div v-if="line.trText" class="lyric-translation" :style="{ fontSize: `${fontSize * 0.6}px` }">
                {{ line.trText }}
              </div>
            </div>
          </template>
          <div v-else class="lyric-empty">暂无歌词</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, nextTick, onMounted, onUnmounted, ref, watch } from 'vue';

defineOptions({
  name: 'Lyric',
});

const windowData = window as any;
const containerRef = ref<HTMLElement | null>(null);
const containerHeight = ref(0);
const lineHeight = ref(60);
const currentIndex = ref(0);
const isInitialized = ref(false);
// 字体大小控制
const fontSize = ref(24); // 默认字体大小
const fontSizeStep = 2; // 每次整的步长

// 静态数据
const staticData = ref<{
  lrcArray: Array<{ text: string; trText: string }>;
  lrcTimeArray: number[];
  allTime: number;
}>({
  lrcArray: [],
  lrcTimeArray: [],
  allTime: 0,
});

// 动态数据
const dynamicData = ref({
  nowTime: 0,
  startCurrentTime: 0,
  nextTime: 0,
  isPlay: true,
});

const lyricSetting = ref({
  ...(localStorage.getItem('lyricData')
    ? JSON.parse(localStorage.getItem('lyricData') || '')
    : {
        isTop: false,
        theme: 'dark',
        isLock: false,
      }),
});

let hideControlsTimer: number | null = null;

const isHovering = ref(false);

// 计算是否栏
const showControls = computed(() => {
  if (lyricSetting.value.isLock) {
    return isHovering.value;
  }
  return true;
});

// 清除隐藏定时器
const clearHideTimer = () => {
  if (hideControlsTimer) {
    clearTimeout(hideControlsTimer);
    hideControlsTimer = null;
  }
};

// 处理鼠标进入窗口
const handleMouseEnter = () => {
  if (!lyricSetting.value.isLock) return;
  isHovering.value = true;
};

// 处理鼠标离开窗口
const handleMouseLeave = () => {
  if (!lyricSetting.value.isLock) return;
  isHovering.value = false;
};

// 监听锁定状态变化
watch(
  () => lyricSetting.value.isLock,
  (newLock: boolean) => {
    if (newLock) {
      isHovering.value = false;
    }
  },
);

onMounted(() => {
  // 初始化时，如果是锁定状态，确保控制栏隐藏
  if (lyricSetting.value.isLock) {
    isHovering.value = false;
  }
});

onUnmounted(() => {
  clearHideTimer();
});

// 计算歌词滚动位置
const wrapperStyle = computed(() => {
  if (!isInitialized.value || !containerHeight.value) {
    return {
      transform: 'translateY(0)',
      transition: 'none',
    };
  }

  // 计算容器中心点
  const containerCenter = containerHeight.value / 2;

  // 计算当前行到顶部的距离（包含padding）
  const currentLineTop = currentIndex.value * lineHeight.value + containerHeight.value * 0.2; // 加上顶部padding

  // 计算偏移量，使当前行居中
  const targetOffset = containerCenter - currentLineTop;

  // 计算内容总高度（包含padding）
  const contentHeight = staticData.value.lrcArray.length * lineHeight.value + containerHeight.value * 0.4; // 上下padding各20vh

  // 计算最小和最大偏移量
  const minOffset = -(contentHeight - containerHeight.value);
  const maxOffset = 0;

  // 限制偏移量在合理范围内
  const finalOffset = Math.min(maxOffset, Math.max(minOffset, targetOffset));

  return {
    transform: `translateY(${finalOffset}px)`,
    transition: isInitialized.value ? 'transform 0.3s cubic-bezier(0.4, 0, 0.2, 1)' : 'none',
  };
});

const lyricLineStyle = computed(() => ({
  height: `${lineHeight.value}px`,
}));
// 更新容器高度和行高
const updateContainerHeight = () => {
  if (!containerRef.value) return;

  // 更新容器高度
  containerHeight.value = containerRef.value.clientHeight;

  // 计算基础行高(字体大小的2.5倍)
  const baseLineHeight = fontSize.value * 2.5;

  // 计算最大允许行高(容器高度的1/4)
  const maxAllowedHeight = containerHeight.value / 3;

  // 设置行高(不小于40px,不大于最大允许高度)
  lineHeight.value = Math.min(maxAllowedHeight, Math.max(40, baseLineHeight));
};

// 处理字体大小变化
const handleFontSizeChange = async () => {
  // 先保存字体大小
  saveFontSize();

  // 更新容器高度和行高
  updateContainerHeight();
};

// 增加字体大小
const increaseFontSize = async () => {
  if (fontSize.value < 48) {
    fontSize.value += fontSizeStep;
    await handleFontSizeChange();
  }
};

// 减小字体大小
const decreaseFontSize = async () => {
  if (fontSize.value > 12) {
    fontSize.value -= fontSizeStep;
    await handleFontSizeChange();
  }
};

// 保存字体大小到本地存储
const saveFontSize = () => {
  localStorage.setItem('lyricFontSize', fontSize.value.toString());
};

// 监听容器大小变化
onMounted(() => {
  const resizeObserver = new ResizeObserver(() => {
    updateContainerHeight();
  });

  if (containerRef.value) {
    resizeObserver.observe(containerRef.value);
  }

  onUnmounted(() => {
    resizeObserver.disconnect();
  });
});

// 动画帧ID
const animationFrameId = ref<number | null>(null);

// 实际播放时间
const actualTime = ref(0);

// 计算当前行的进度
const currentProgress = computed(() => {
  const { startCurrentTime, nextTime, isPlay } = dynamicData.value;
  if (!startCurrentTime || !nextTime || !isPlay) return 0;

  const duration = nextTime - startCurrentTime;
  const elapsed = actualTime.value - startCurrentTime;
  return Math.min(Math.max(elapsed / duration, 0), 1);
});

// 获取歌词样式
const getLyricStyle = (index: number) => {
  if (index !== currentIndex.value) return {};

  const progress = currentProgress.value * 100;
  return {
    background: `linear-gradient(to right, var(--highlight-color) ${progress}%, var(--text-color) ${progress}%)`,
    WebkitBackgroundClip: 'text',
    WebkitTextFillColor: 'transparent',
    transition: 'all 0.1s linear',
  };
};

// 时间偏移量（毫秒）
const TIME_OFFSET = 400;

// 更新动画
const updateProgress = () => {
  if (!dynamicData.value.isPlay) {
    if (animationFrameId.value) {
      cancelAnimationFrame(animationFrameId.value);
      animationFrameId.value = null;
    }
    return;
  }

  // 计算实际时间，添加偏移量
  const timeDiff = (performance.now() - lastUpdateTime.value) / 1000;
  actualTime.value = dynamicData.value.nowTime + timeDiff + TIME_OFFSET / 1000;

  // 继续动画
  animationFrameId.value = requestAnimationFrame(updateProgress);
};

// 记录上次更新时间
const lastUpdateTime = ref(performance.now());

// 监听数据更新
watch(
  () => dynamicData.value,
  (newData: any) => {
    // 更新最后更新时间
    lastUpdateTime.value = performance.now();

    // 更新实际时间，包含偏移量
    actualTime.value = newData.nowTime + TIME_OFFSET / 1000;

    // 如果正在播放且没有动画，启动动画
    if (newData.isPlay && !animationFrameId.value) {
      updateProgress();
    }
  },
  { deep: true },
);

// 监听播放状态变化
watch(
  () => dynamicData.value.isPlay,
  (isPlaying: boolean) => {
    if (isPlaying) {
      lastUpdateTime.value = performance.now();
      updateProgress();
    } else if (animationFrameId.value) {
      cancelAnimationFrame(animationFrameId.value);
      animationFrameId.value = null;
    }
  },
);

// 修改数据更新处理
const handleDataUpdate = (parsedData: {
  nowTime: number;
  startCurrentTime: number;
  nextTime: number;
  isPlay: boolean;
  nowIndex: number;
}) => {
  // 确保数据存在且格式正确
  if (!parsedData || typeof parsedData.nowTime !== 'number') {
    console.error('Invalid update data received:', parsedData);
    return;
  }

  dynamicData.value = {
    nowTime: parsedData.nowTime,
    startCurrentTime: parsedData.startCurrentTime,
    nextTime: parsedData.nextTime,
    isPlay: parsedData.isPlay,
  };

  // 更新索引
  if (typeof parsedData.nowIndex === 'number' && parsedData.nowIndex !== currentIndex.value) {
    currentIndex.value = parsedData.nowIndex;
  }
};

onMounted(() => {
  // 加载保存的字体大小
  const savedFontSize = localStorage.getItem('lyricFontSize');
  if (savedFontSize) {
    fontSize.value = Number(savedFontSize);
    lineHeight.value = fontSize.value * 2.5;
  }

  // 初始化容器高度
  updateContainerHeight();
  window.addEventListener('resize', updateContainerHeight);

  // 监听歌词数据
  windowData.electron.ipcRenderer.on('receive-lyric', (data: string) => {
    try {
      const parsedData = JSON.parse(data);
      if (parsedData.type === 'init') {
        // 初始化重置状态
        currentIndex.value = 0;
        isInitialized.value = false;

        // 清理可能存在的动画
        if (animationFrameId.value) {
          cancelAnimationFrame(animationFrameId.value);
          animationFrameId.value = null;
        }

        // 保据格式正确
        if (Array.isArray(parsedData.lrcArray)) {
          staticData.value = {
            lrcArray: parsedData.lrcArray,
            lrcTimeArray: parsedData.lrcTimeArray || [],
            allTime: parsedData.allTime || 0,
          };
        } else {
          console.error('Invalid lyric array format:', parsedData);
        }
        nextTick(() => {
          isInitialized.value = true;
        });
      } else if (parsedData.type === 'update') {
        handleDataUpdate(parsedData);
      }
    } catch (error) {
      console.error('Error parsing lyric data:', error);
    }
  });
});

onUnmounted(() => {
  window.removeEventListener('resize', updateContainerHeight);
});

const checkTheme = () => {
  if (lyricSetting.value.theme === 'light') {
    lyricSetting.value.theme = 'dark';
  } else {
    lyricSetting.value.theme = 'light';
  }
};

const handleTop = () => {
  lyricSetting.value.isTop = !lyricSetting.value.isTop;
  windowData.electron.ipcRenderer.send('top-lyric', lyricSetting.value.isTop);
};

const handleLock = () => {
  lyricSetting.value.isLock = !lyricSetting.value.isLock;
};

const handleClose = () => {
  windowData.electron.ipcRenderer.send('close-lyric');
};

watch(
  () => lyricSetting.value,
  (newValue: any) => {
    localStorage.setItem('lyricData', JSON.stringify(newValue));
  },
  { deep: true },
);
</script>

<style>
body {
  background-color: transparent !important;
}
</style>

<style lang="scss" scoped>
.lyric-window {
  width: 100vw;
  height: 100vh;
  position: relative;
  overflow: hidden;
  background: transparent;

  &.dark {
    --bg-color: transparent;
    --text-color: #ffffff;
    --text-secondary: rgba(255, 255, 255, 0.6);
    --highlight-color: #1db954;
    --control-bg: rgba(0, 0, 0, 0.3);
  }

  &.light {
    --bg-color: transparent;
    --text-color: #333333;
    --text-secondary: rgba(51, 51, 51, 0.6);
    --highlight-color: #1db954;
    --control-bg: rgba(255, 255, 255, 0.3);
  }

  &.lyric_lock {
    .control-bar {
      background: var(--control-bg);

      &-show {
        opacity: 1;
      }
    }
  }
}

.control-bar {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 40px;
  background: var(--control-bg);
  backdrop-filter: blur(8px);
  display: flex;
  justify-content: flex-end;
  align-items: center;
  padding: 0 20px;
  opacity: 0;
  visibility: hidden;
  transition:
    opacity 0.2s ease,
    visibility 0.2s ease;
  -webkit-app-region: drag;
  z-index: 100;

  &-show {
    opacity: 1;
    visibility: visible;
  }

  .font-size-controls {
    margin-right: auto; // 将字体控制放在侧
    padding-right: 20px;
    -webkit-app-region: no-drag;

    .n-button {
      i {
        font-size: 16px;
      }
    }
  }

  .control-buttons {
    -webkit-app-region: no-drag;
  }
}

.control-buttons {
  display: flex;
  gap: 16px;
  -webkit-app-region: no-drag;
}

.control-button {
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  border-radius: 50%;
  color: var(--text-color);
  transition: all 0.2s ease;
  backdrop-filter: blur(4px);

  &:hover {
    background: var(--control-bg);
  }

  i {
    font-size: 18px;
    text-shadow: 0 0 10px rgba(0, 0, 0, 0.3);

    &.active {
      color: var(--highlight-color);
    }
  }
}

.lyric-container {
  position: absolute;
  top: 40px;
  left: 0;
  right: 0;
  bottom: 0;
  overflow: hidden;
}

.lyric-scroll {
  height: 100%;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  align-items: center;
  mask-image: linear-gradient(to bottom, transparent 0%, black 20%, black 80%, transparent 100%);
}

.lyric-wrapper {
  will-change: transform;
  padding: 20vh 0;
  transform-origin: center center;
  transition: transform 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.lyric-line {
  padding: 4px 20px;
  text-align: center;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;

  &.lyric-line-current {
    transform: scale(1.05);
    opacity: 1;
  }

  &.lyric-line-passed,
  &.lyric-line-next {
    opacity: 0.6;
  }
}

.lyric-text {
  font-weight: 600;
  margin-bottom: 2px;
  color: var(--text-color);
  white-space: pre-wrap;
  word-break: break-all;
  text-shadow: 0 0 10px rgba(0, 0, 0, 0.3);
  transition: all 0.2s ease;
  line-height: 1.4;
}

.lyric-translation {
  color: var(--text-secondary);
  white-space: pre-wrap;
  word-break: break-all;
  text-shadow: 0 0 10px rgba(0, 0, 0, 0.3);
  transition: font-size 0.2s ease;
  line-height: 1.4; // 添加行高比例
}

.lyric-empty {
  text-align: center;
  color: var(--text-secondary);
  font-size: 16px;
  padding: 20px;
}

body {
  background-color: transparent !important;
  margin: 0;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans',
    'Helvetica Neue', sans-serif;
}

.lyric-content {
  transition: font-size 0.2s ease;
}

.lyric-line-current {
  opacity: 1;
}
</style>
