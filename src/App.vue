<script setup lang="ts">
import Kaleidoscope from './components/Kaleidoscope.vue';
import { computed, ref, useTemplateRef } from 'vue';
import { ScopeShape } from './scopeShape.ts';
import CaptureModal from './components/CaptureModal.vue';
import TimeSelect from './components/TimeSelect.vue';

const saveNextFrame = ref(false);
const capturedFrame = ref(null as null|string);
const showCapture = ref(false);
const selectedIndex = ref(ScopeShape.Square);
const scopeAutoRotationVelocity = ref(0);
const uploadedImages = ref([] as string[]);
const fileInputRef = useTemplateRef('file-input');
const hasStarted = ref(false);

type RemovedImage = { image: string; index: number };
const undoStack = ref<RemovedImage[]>([]);
const canUndo = computed(() => undoStack.value.length > 0);

const isOnGallery = ref(false);
const activeOrbsCount = ref(0);
const hasGalleryItems = computed(() => undoStack.value.length > 0 || uploadedImages.value.length > 0);

const handleGalleryViewChange = (onGallery: boolean) => {
  isOnGallery.value = onGallery;
};

const handleActiveOrbsChange = (count: number) => {
  activeOrbsCount.value = count;
};

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

const handleRemoveUploadedImage = (removeIndex: number) => {
  const image = uploadedImages.value[removeIndex];
  if (image === undefined) {
    return;
  }
  undoStack.value.push({ image, index: removeIndex });
  uploadedImages.value = uploadedImages.value.filter((_, i) => i !== removeIndex);
};

const undoLastRemoval = () => {
  const last = undoStack.value.pop();
  if (!last) {
    return;
  }
  const next = [...uploadedImages.value];
  const insertIndex = Math.min(Math.max(last.index, 0), next.length);
  next.splice(insertIndex, 0, last.image);
  uploadedImages.value = next;
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
  <TimeSelect v-if="!hasStarted" @start="hasStarted = true" />

  <template v-else>
    <Kaleidoscope
      :scope-shape="selectedIndex"
      :scope-auto-rotation-velocity="scopeAutoRotationVelocity"
      :save-next-frame="saveNextFrame"
      :uploaded-images="uploadedImages"
      @save-frame="savedFrame"
      @upload-click="handleUploadClick"
      @remove-uploaded-image="handleRemoveUploadedImage"
      @gallery-view-change="handleGalleryViewChange"
      @active-orbs-change="handleActiveOrbsChange"
    />

    <button
      v-if="canUndo && !isOnGallery"
      type="button"
      class="fixed left-1 top-1 z-[300] text-black active:scale-95 transition hover:opacity-70 p-4 cursor-pointer"
      @click.stop="undoLastRemoval"
      aria-label="Undo"
    >
      <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
        <path d="M9 14 4 9l5-5"/>
        <path d="M4 9h10.5a5.5 5.5 0 0 1 5.5 5.5v0a5.5 5.5 0 0 1-5.5 5.5H11"/>
      </svg>
    </button>
    <div
      v-if="!showCapture && activeOrbsCount === 0 && !hasGalleryItems"
      class="fixed left-1/2 pointer-events-none z-10 min-w-[200px] min-h-[200px] w-[60vmin] h-[60vmin] -translate-x-1/2 -translate-y-1/2 flex items-center justify-center rounded-full border-2 border-black"
      style="top: 50%"
      aria-hidden="true"
    >
      <img
        src="/upload.svg"
        alt=""
        class="w-12 h-12 sm:w-14 sm:h-14 select-none brightness-0"
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
</template>

<style scoped>
</style>
