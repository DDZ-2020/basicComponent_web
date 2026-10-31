<template>
  <ElDialog
    v-model="dialogVisible"
    width="80%"
    top="6vh"
    draggable
    destroy-on-close
    custom-class="image-preview-dialog"
    :title="currentTitle"
    @closed="resetView"
  >
    <div class="preview-shell">
      <div class="preview-body">
        <div
          ref="imageStageRef"
          class="image-stage"
          @wheel.prevent="handleWheel"
          @pointerdown="handlePointerDown"
          @pointermove="handlePointerMove"
          @pointerup="handlePointerUp"
          @pointerleave="handlePointerUp"
          @pointercancel="handlePointerUp"
        >
          <button
            v-if="imageCount > 0"
            class="nav-btn nav-btn-left"
            @click.stop="prevImage"
            @pointerdown.stop
            title="上一张"
          >
            <ElIcon><ArrowLeft /></ElIcon>
          </button>

          <img
            :src="currentImageUrl"
            alt="preview"
            class="preview-image"
            :style="imageStyle"
            draggable="false"
          />

          <button
            v-if="imageCount > 0"
            class="nav-btn nav-btn-right"
            @click.stop="nextImage"
            @pointerdown.stop
            title="下一张"
          >
            <ElIcon><ArrowRight /></ElIcon>
          </button>
        </div>
      </div>

      <div class="preview-toolbar">
        <div class="toolbar-group right">
          <button
            v-if="imageCount > 0"
            class="toolbar-btn"
            @click.stop="prevImage"
            @pointerdown.stop
            title="上一张"
          >
            <ElIcon><ArrowLeft /></ElIcon>
          </button>
          <span class="count-display scale-display">{{ currentIndex + 1 }}/{{ imageCount }}</span>
          <button
            v-if="imageCount > 0"
            class="toolbar-btn"
            @click.stop="nextImage"
            @pointerdown.stop
            title="下一张"
          >
            <ElIcon><ArrowRight /></ElIcon>
          </button>
          <div class="line"></div>
          <button class="toolbar-btn" @click="zoomOut" title="缩小">
            <ElIcon><ZoomOut /></ElIcon>
          </button>
          <span class="scale-display">{{ scaleDisplay }}</span>
          <button class="toolbar-btn" @click="zoomIn" title="放大">
            <ElIcon><ZoomIn /></ElIcon>
          </button>
          <button class="toolbar-btn" @click="rotateLeft" title="左旋转">
            <ElIcon><RefreshLeft /></ElIcon>
          </button>
          <button class="toolbar-btn" @click="rotateRight" title="右旋转">
            <ElIcon><RefreshRight /></ElIcon>
          </button>
          <button class="toolbar-btn" @click="toggleOriginalSize" title="显示原始尺寸">
            <ElIcon>
              <component :is="currentSizeIcon" />
            </ElIcon>
          </button>
          <button class="toolbar-btn" @click="downloadImage" title="下载">
            <ElIcon><Download /></ElIcon>
          </button>
        </div>
      </div>
    </div>
  </ElDialog>
</template>

<script setup>
import { computed, ref, reactive } from 'vue'
import { ElDialog, ElIcon, ElMessage } from 'element-plus'
import {
  ArrowLeft,
  ArrowRight,
  ZoomIn,
  ZoomOut,
  RefreshLeft,
  RefreshRight,
  FullScreen,
  ScaleToOriginal,
  Download,
} from '@element-plus/icons-vue'

const props = defineProps({
  modelValue: {
    type: Boolean,
    default: false,
  },
  imageUrl: {
    type: String,
    default: 'https://images.unsplash.com/photo-1500530855697-b586d89ba3ee?auto=format&fit=crop&w=1200&q=80',
  },
  imageList: {
    type: Array,
    default: () => [],
  },
  imageInfo: {
    type: Object,
    default: () => ({})
  },
  imageInfoList: {
    type: Array,
    default: () => [
        {
            imageName: '示例图片1',
            imageUrl: 'https://images.unsplash.com/photo-1500530855697-b586d89ba3ee?auto=format&fit=crop&w=1200&q=80',
        },
        {
            imageName: '示例图片2',
            imageUrl: 'https://images.unsplash.com/photo-1516117172878-fd2c41f4a759?auto=format&fit=crop&w=1200&q=80',  
        },
        {
            imageUrl: 'https://images.unsplash.com/photo-1522205408450-add114ad53fe?auto=format&fit=crop&w=1200&q=80',  
        },
    ]
  },
})

const emit = defineEmits(['update:modelValue'])

const dialogVisible = computed({
  get: () => props.modelValue,
  set: (value) => emit('update:modelValue', value),
})

const scale = ref(1)
const rotate = ref(0)
const showOriginalSize = ref(false)
const imageStageRef = ref(null)
const dragState = reactive({
  active: false,
  startX: 0,
  startY: 0,
  originX: 0,
  originY: 0,
})

const offset = reactive({ x: 0, y: 0 })
const minScale = 0.2  // 缩小的最小比例
const maxScale = 3  // 放大的最大比例
const currentIndex = ref(0)

// 图片列表标准化
const normalizedImageList = computed(() => {
  if (props.imageInfoList && props.imageInfoList.length) {
    return props.imageInfoList.map((item) => ({
      imageName: item && item.imageName ? item.imageName : '',
      imageUrl: item && item.imageUrl ? item.imageUrl : '',
    }))
  }

  if (props.imageList && props.imageList.length) {
    return props.imageList.map((item) => ({
      imageName: '',
      imageUrl: item,
    }))
  }

  if (props.imageInfo && props.imageInfo.imageUrl) {
    return [{
      imageName: props.imageInfo.imageName || '',
      imageUrl: props.imageInfo.imageUrl,
    }]
  }

  return [{
    imageName: '',
    imageUrl: props.imageUrl,
  }]
})

const imageCount = computed(() => normalizedImageList.value.length)
const currentImageUrl = computed(() => {
  const current = normalizedImageList.value[currentIndex.value] || normalizedImageList.value[0]
  return current ? current.imageUrl : props.imageUrl
})
const scaleDisplay = computed(() => `${Math.round(scale.value * 100)}%`)

const imageStyle = computed(() => ({
  transform: `translate(${offset.x}px, ${offset.y}px) translate(-50%, -50%) scale(${scale.value}) rotate(${rotate.value}deg)`,
  width: showOriginalSize.value ? 'auto' : '100%',
  maxWidth: showOriginalSize.value ? 'none' : '100%',
  maxHeight: showOriginalSize.value ? 'none' : '70vh',
}))

// 约束范围
const clamp = (value, min, max) => Math.min(Math.max(value, min), max)
const zoomBy = (delta) => {
  scale.value = clamp(scale.value + delta, minScale, maxScale)
}
const zoomIn = () => zoomBy(0.1)
const zoomOut = () => zoomBy(-0.1)

const rotateLeft = () => {
  rotate.value -= 90
}

const rotateRight = () => {
  rotate.value += 90
}

const resetView = () => {
  scale.value = 1
  rotate.value = 0
  showOriginalSize.value = false
  offset.x = 0
  offset.y = 0
}

const toggleOriginalSize = () => {
  showOriginalSize.value = !showOriginalSize.value
  if (showOriginalSize.value) {
    scale.value = 1
  }
}

const prevImage = () => {
  if (imageCount.value === 0) return
  currentIndex.value = (currentIndex.value - 1 + imageCount.value) % imageCount.value
  resetView()
}

const nextImage = () => {
  if (imageCount.value === 0) return
  currentIndex.value = (currentIndex.value + 1) % imageCount.value
  resetView()
}

const handleWheel = (event) => {
  event.preventDefault()
  if (event.deltaY < 0) {
    zoomIn()
  } else {
    zoomOut()
  }
}

// 鼠标按下，拖拽开始，记录起点坐标
const handlePointerDown = (event) => {
  if (event.button !== 0) return
  dragState.active = true
  dragState.startX = event.clientX
  dragState.startY = event.clientY
  dragState.originX = offset.x
  dragState.originY = offset.y
  // DOM存在的时候执行，防止还没渲染好，DOM为null报错
  if (imageStageRef.value) {
    imageStageRef.value.setPointerCapture?.(event.pointerId)
  }
}

// 鼠标移动，实时计算偏移，拖动图片平移
const handlePointerMove = (event) => {
  if (!dragState.active) return
  offset.x = dragState.originX + (event.clientX - dragState.startX)
  offset.y = dragState.originY + (event.clientY - dragState.startY)
}

// 鼠标左键收起，拖拽结束
const handlePointerUp = (event) => {
  if (imageStageRef.value && dragState.active) {
    imageStageRef.value.releasePointerCapture?.(event.pointerId)
  }
  dragState.active = false
}

const downloadImage = async () => {
  try {
    const response = await fetch(currentImageUrl.value)
    const blob = await response.blob()
    const url = URL.createObjectURL(blob)
    const link = document.createElement('a')
    link.href = url
    link.download = 'preview-image.png'
    link.click()
    URL.revokeObjectURL(url)
    ElMessage.success('图片下载成功')
  } catch (error) {
    ElMessage.error('图片下载失败')
  }
}

const closePreview = () => {
  dialogVisible.value = false
  resetView()
}

const currentSizeIcon = computed(() => (showOriginalSize.value ? ScaleToOriginal : FullScreen))
const currentTitle = computed(() => {
  const list = normalizedImageList.value
  if (list.length) {
    const current = list[currentIndex.value] || list[0]
    if (current && current.imageName) return current.imageName
    return '图片预览'
  }
  return '图片预览'
})
</script>

<style lang="less" scoped>
.image-preview-dialog {
  border-radius: 18px;
  overflow: hidden;
  background: rgba(255, 255, 255, 0.9);
  backdrop-filter: blur(8px);

  :deep(.el-dialog__header) {
    display: none;
  }

  :deep(.el-dialog__body) {
    padding: 0;
    background: #f5f5f5;
  }
}

.preview-shell {
  display: flex;
  flex-direction: column;
  background: #f3f3f3;
}

.preview-body {
  width: 100%;
  height: 70vh;
  min-height: 500px;
  background: #f3f3f3;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
}

.image-stage {
  position: relative;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
  background: rgba(0, 0, 0, 0.25);
  cursor: grab;
  user-select: none;
  touch-action: none;

  &:active {
    cursor: grabbing;
  }
}

.preview-image {
  position: absolute;
  top: 50%;
  left: 50%;
  object-fit: contain;
  transition: transform 0.2s ease;
  user-select: none;
  max-width: 100%;
  max-height: 100%;
}

.preview-toolbar {
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 18px 24px 20px;
  background: rgba(255, 255, 255, 0.75);
  border-top: 1px solid rgba(0, 0, 0, 0.06);
}

.toolbar-group {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
}

.toolbar-btn {
  width: 40px;
  height: 40px;
  border: 1px solid rgba(0, 0, 0, 0.08);
  background: #fff;
  border-radius: 50%;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.2s ease;
  color: #333;

  &:hover {
    transform: translateY(-1px);
    box-shadow: 0 6px 16px rgba(0, 0, 0, 0.08);
  }
}

.line {
  height: 30px;
  width: 1px;
  display: inline-block;
  background: #ccc;
}

.scale-display {
  min-width: 56px;
  text-align: center;
  font-size: 14px;
  color: #333;
  font-weight: 600;
}

.nav-btn {
  position: absolute;
  top: 50%;
  z-index: 2;
  width: 40px;
  height: 40px;
  border: 1px solid rgba(0, 0, 0, 0.08);
  background: rgba(255, 255, 255, 0.8);
  border-radius: 50%;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  color: #333;
  transform: translateY(-50%);

  &:hover {
    box-shadow: 0 6px 16px rgba(0, 0, 0, 0.08);
  }
}

.nav-btn-left {
  left: 24px;
}

.nav-btn-right {
  right: 24px;
}
</style>
