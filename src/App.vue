<script setup lang="ts">
import Kaleidoscope from './components/Kaleidoscope.vue';
import { ref, useTemplateRef } from 'vue';
import { ScopeShape } from './scopeShape.ts';
import CaptureModal from './components/CaptureModal.vue';

const saveNextFrame = ref(false);
const capturedFrame = ref(null as null|string);
const showCapture = ref(false);
const selectedIndex = ref(ScopeShape.Square);
const scopeAutoRotationVelocity = ref(0);
const uploadedImages = ref([] as string[]);
const fileInputRef = useTemplateRef('file-input');

function savedFrame(objectUrl: string) {
  saveNextFrame.value = false;
  showCapture.value = true;
  capturedFrame.value = objectUrl;
}

const handleFileUpload = (event: Event) => {
  const target = event.target as HTMLInputElement;
  const files = target.files;
  if (files && files.length > 0) {
    const newImages: string[] = [];
    let loadedCount = 0;
    
    Array.from(files).forEach((file) => {
      if (file.type.startsWith('image/')) {
        const reader = new FileReader();
        reader.onload = (e) => {
          const result = e.target?.result;
          if (typeof result === 'string') {
            newImages.push(result);
            loadedCount++;
            if (loadedCount === Array.from(files).filter(f => f.type.startsWith('image/')).length) {
              uploadedImages.value = [...uploadedImages.value, ...newImages];
            }
          }
        };
        reader.readAsDataURL(file);
      }
    });
  }
};

const handleUploadClick = () => {
  fileInputRef.value?.click();
};

window.addEventListener('keypress', (keyEvent) => {
  if (keyEvent.code === 'Digit1') {
    selectedIndex.value = ScopeShape.Equilateral;
  } else if (keyEvent.code === 'Digit2') {
    selectedIndex.value = ScopeShape.Square;
  }
});

</script>

<template>
  <Kaleidoscope
    :scope-shape="selectedIndex"
    :scope-auto-rotation-velocity="scopeAutoRotationVelocity"
    :save-next-frame="saveNextFrame"
    :uploaded-images="uploadedImages"
    @save-frame="savedFrame"
    @upload-click="handleUploadClick"
  />
  <div
    v-if="!showCapture && uploadedImages.length === 0"
    class="fixed left-1/2 pointer-events-none z-10 min-w-[200px] min-h-[200px] w-[80vmin] h-[80vmin] -translate-x-1/2 -translate-y-1/2 flex items-center justify-center"
    style="top: 50%"
    aria-hidden="true"
  >
    <img
      src="/upload.svg"
      alt=""
      class="w-12 h-12 sm:w-14 sm:h-14 opacity-70 dark:opacity-60 select-none"
    >
  </div>
  <input
    ref="file-input"
    type="file"
    accept="image/*"
    multiple
    class="hidden"
    @change="handleFileUpload"
  >
  <div
    v-if="showCapture && capturedFrame !== null"
    class="absolute left-0 top-0 h-screen w-screen grid grid-cols-1 grid-rows-1 p-2 pb-4 overflow-auto bg-black/20 backdrop-blur-xl"
    style="width:100dvw;height:100dvh;background-size: cover;"
    @click="showCapture = false"
  >
    <CaptureModal
      :url="capturedFrame"
      @done="showCapture = false"
    />
  </div>
</template>

<style scoped>
</style>
