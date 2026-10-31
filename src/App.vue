<script setup>
import { ref } from 'vue'
import { ElButton } from 'element-plus'
import ImagePreview from './components/preview/ImagePreview.vue'

const previewVisible = ref(true)
const previewImage = ref('https://images.unsplash.com/photo-1500530855697-b586d89ba3ee?auto=format&fit=crop&w=1200&q=80')

const openPreview = () => {
  previewVisible.value = true
}
</script>

<template>
  <div class="page-shell">
    <div class="demo-card">
      <ElButton type="primary" @click="openPreview">打开图片预览</ElButton>
    </div>

    <ImagePreview
      v-model="previewVisible"
      :image-url="previewImage"
    />
  </div>
</template>

<style lang="less" scoped>
.page-shell {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(120deg, #f5f7fa, #e9eef6);
}

.demo-card {
  padding: 24px 32px;
  border-radius: 16px;
  background: rgba(255, 255, 255, 0.8);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.08);
}
</style>
