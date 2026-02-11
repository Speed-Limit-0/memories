<script setup lang="ts">
import Kaleidoscope from './components/Kaleidoscope.vue';
import { computed, ref, useTemplateRef } from 'vue';
import { ScopeShape } from './scopeShape.ts';
import CaptureModal from './components/CaptureModal.vue';

const saveNextFrame = ref(false);
const capturedFrame = ref(null as null|string);
const showCapture = ref(false);
const selectedIndex = ref(ScopeShape.Square);
const scopeAutoRotationVelocity = ref(0);
const uploadedImages = ref([] as string[]);
const fileInputRef = useTemplateRef('file-input');

type RemovedImage = { image: string; index: number; type: 'removal' };
type GalleryStatusChange = { galleryId: string; previousStatus: 'active' | 'left' | 'right'; type: 'status' };
type UndoAction = RemovedImage | GalleryStatusChange;
const undoStack = ref<UndoAction[]>([]);
const canUndo = computed(() => undoStack.value.length > 0);

const isOnGallery = ref(false);
const activeOrbsCount = ref(0);
const currentOrbIdx = ref(0);
const totalGalleryCount = ref(0);
const hasGalleryItems = computed(() => undoStack.value.length > 0 || uploadedImages.value.length > 0);

// Computed: image counter display ("1 of 9")
const currentImageNumber = computed(() => {
  if (totalGalleryCount.value === 0) return 0;
  const processed = totalGalleryCount.value - activeOrbsCount.value;
  return processed + currentOrbIdx.value + 1;
});

// Computed: date display (placeholder using current month/year)
const currentDateDisplay = computed(() => {
  const months = ['January', 'February', 'March', 'April', 'May', 'June',
                  'July', 'August', 'September', 'October', 'November', 'December'];
  const now = new Date();
  return `${months[now.getMonth()]} ${now.getFullYear()}`;
});

// Show the top bar when there are images (active or gallery)
const showTopBar = computed(() => {
  return activeOrbsCount.value > 0 || totalGalleryCount.value > 0;
});

// Show counter and bottom buttons only during active orb swiping
const showOrbUI = computed(() => {
  return activeOrbsCount.value > 0 && !isOnGallery.value;
});

const handleGalleryViewChange = (onGallery: boolean) => {
  isOnGallery.value = onGallery;
};

const handleActiveOrbsChange = (count: number) => {
  activeOrbsCount.value = count;
};

const handleCenteredIndexChange = (index: number) => {
  currentOrbIdx.value = index;
};

const handleTotalGalleryCountChange = (count: number) => {
  totalGalleryCount.value = count;
};

function savedFrame(objectUrl: string) {
  saveNextFrame.value = false;
  showCapture.value = true;
  capturedFrame.value = objectUrl;
}

const handleFileUpload = async (event: Event) => {
  const target = event.target as HTMLInputElement;
  const files = target.files;
  if (files && files.length > 0) {
    const imageFiles = Array.from(files).filter(f => f.type.startsWith('image/'));
    
    if (imageFiles.length === 0) {
      // Reset input
      if (target) target.value = '';
      return;
    }
    
    // Load all images in parallel with proper error handling
    const loadPromises = imageFiles.map((file) => {
      return new Promise<string>((resolve, reject) => {
        const reader = new FileReader();
        reader.onload = (e) => {
          const result = e.target?.result;
          if (typeof result === 'string') {
            resolve(result);
          } else {
            reject(new Error('Failed to read file as data URL'));
          }
        };
        reader.onerror = () => {
          reject(new Error(`Failed to read file: ${file.name}`));
        };
        reader.readAsDataURL(file);
      });
    });
    
    try {
      const newImages = await Promise.all(loadPromises);
      // Append new images to existing ones
      uploadedImages.value = [...uploadedImages.value, ...newImages];
    } catch (error) {
      console.error('Error loading images:', error);
      // Still try to add any successfully loaded images
    } finally {
      // Reset input so same files can be selected again
      if (target) target.value = '';
    }
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
  undoStack.value.push({ image, index: removeIndex, type: 'removal' });
  uploadedImages.value = uploadedImages.value.filter((_, i) => i !== removeIndex);
};

const handleGalleryStatusChange = (galleryId: string, previousStatus: 'active' | 'left' | 'right') => {
  undoStack.value.push({ galleryId, previousStatus, type: 'status' });
};

// Close/reset: return to upload screen
const handleClose = () => {
  uploadedImages.value = [];
  undoStack.value = [];
  currentOrbIdx.value = 0;
  totalGalleryCount.value = 0;
  if (kaleidoscopeRef.value) {
    (kaleidoscopeRef.value as any).clearGallery();
  }
};

// Bottom button handlers
const handleAISlop = () => {
  if (kaleidoscopeRef.value && activeOrbsCount.value > 0) {
    (kaleidoscopeRef.value as any).dismissCurrentOrb(-1); // Fly left → AI Slop (X)
  }
};

const handleRealMemory = () => {
  if (kaleidoscopeRef.value && activeOrbsCount.value > 0) {
    (kaleidoscopeRef.value as any).dismissCurrentOrb(1); // Fly right → Real Memory (checkmark)
  }
};

const kaleidoscopeRef = useTemplateRef('kaleidoscope');

const undoLastRemoval = () => {
  const last = undoStack.value.pop();
  if (!last) {
    return;
  }
  
  if (last.type === 'removal') {
    // Restore removed image
    const next = [...uploadedImages.value];
    const insertIndex = Math.min(Math.max(last.index, 0), next.length);
    next.splice(insertIndex, 0, last.image);
    uploadedImages.value = next;
    
    // Smoothly scroll to the returned orb after a brief delay to allow it to load
    setTimeout(() => {
      if (kaleidoscopeRef.value) {
        (kaleidoscopeRef.value as any).scrollToOrb(insertIndex);
      }
    }, 180);
  } else if (last.type === 'status') {
    // Reset gallery item status to active
    if (kaleidoscopeRef.value) {
      (kaleidoscopeRef.value as any).resetGalleryStatus(last.galleryId);
      // Find the orb index for this gallery item and scroll to it
      setTimeout(() => {
        if (kaleidoscopeRef.value) {
          const orbIndex = (kaleidoscopeRef.value as any).findOrbIndexByGalleryId(last.galleryId);
          if (orbIndex >= 0) {
            (kaleidoscopeRef.value as any).scrollToOrb(orbIndex);
          }
        }
      }, 160);
    }
  }
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
    ref="kaleidoscope"
    :scope-shape="selectedIndex"
    :scope-auto-rotation-velocity="scopeAutoRotationVelocity"
    :save-next-frame="saveNextFrame"
    :uploaded-images="uploadedImages"
    @save-frame="savedFrame"
    @upload-click="handleUploadClick"
    @remove-uploaded-image="handleRemoveUploadedImage"
    @gallery-view-change="handleGalleryViewChange"
    @active-orbs-change="handleActiveOrbsChange"
    @gallery-status-change="handleGalleryStatusChange"
    @centered-index-change="handleCenteredIndexChange"
    @total-gallery-count-change="handleTotalGalleryCountChange"
  />

  <!-- Top Bar: X close, counter, undo -->
  <div
    v-if="showTopBar"
    class="fixed top-0 left-0 right-0 z-[300] flex items-center justify-between"
    style="padding: min(88px, max(44px, 10dvh)) 40px 0 40px; font-family: var(--font-sans);"
  >
    <!-- X Close Button -->
    <button
      @click.stop="handleClose"
      class="w-[30px] h-[30px] rounded-full border border-[#a0a0a0] flex items-center justify-center cursor-pointer active:scale-95 transition shrink-0"
      aria-label="Close"
    >
      <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="#a0a0a0" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
        <path d="M18 6 6 18"/>
        <path d="m6 6 12 12"/>
      </svg>
    </button>

    <!-- Center Counter -->
    <div
      v-if="showOrbUI"
      class="flex flex-col items-center absolute left-1/2 -translate-x-1/2 pointer-events-none"
      :style="{ top: 'min(88px, max(44px, 10dvh))' }"
    >
      <span class="text-[16px] text-black font-light leading-normal" style="letter-spacing: -0.32px;">{{ currentImageNumber }} of {{ totalGalleryCount }}</span>
      <span class="text-[16px] text-[#a0a0a0] font-light leading-normal" style="letter-spacing: -0.32px;">{{ currentDateDisplay }}</span>
    </div>

    <!-- Undo Button -->
    <button
      v-if="canUndo"
      @click.stop="undoLastRemoval"
      class="bg-white border border-[#a0a0a0] rounded-full flex items-center justify-center cursor-pointer active:scale-95 transition shrink-0"
      style="padding: 4px 16px;"
      aria-label="Undo"
    >
      <span class="text-[16px] text-black font-light leading-normal" style="letter-spacing: -0.32px;">Undo</span>
    </button>
    <div v-else class="w-[69px] shrink-0"></div>
  </div>

  <!-- Bottom Buttons: AI Slop (left arrow) + Real Memory (right arrow) -->
  <div
    v-if="showOrbUI"
    class="fixed left-0 right-0 z-[300] flex justify-center gap-[10px]"
    style="bottom: min(104px, max(60px, 12dvh)); padding: 0 40px; font-family: var(--font-sans);"
  >
    <!-- AI Slop Button -->
    <button
      @click.stop="handleAISlop"
      class="flex-1 h-[63px] bg-white border border-[#a0a0a0] rounded-[12px] flex flex-col items-center justify-center gap-[2px] cursor-pointer active:scale-95 transition"
    >
      <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="black" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round">
        <path d="M18 6 6 18"/>
        <path d="m6 6 12 12"/>
      </svg>
      <span class="text-[16px] text-black font-light leading-normal" style="letter-spacing: -0.32px;">AI Slop</span>
    </button>

    <!-- Real Memory Button -->
    <button
      @click.stop="handleRealMemory"
      class="flex-1 h-[63px] bg-white border border-[#a0a0a0] rounded-[12px] flex flex-col items-center justify-center gap-[2px] cursor-pointer active:scale-95 transition"
    >
      <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="black" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round">
        <path d="M20 6 9 17l-5-5"/>
      </svg>
      <span class="text-[16px] text-black font-light leading-normal" style="letter-spacing: -0.32px;">Real Memory</span>
    </button>
  </div>
  <div
    v-if="!showCapture && activeOrbsCount === 0 && !hasGalleryItems"
    class="fixed left-1/2 pointer-events-none z-10 min-w-[200px] min-h-[200px] w-[60vmin] h-[60vmin] -translate-x-1/2 -translate-y-1/2 flex flex-col items-center justify-center"
    style="top: 50%"
    aria-hidden="true"
  >
    <svg
      class="absolute inset-0 w-full h-full"
      viewBox="0 0 100 100"
      xmlns="http://www.w3.org/2000/svg"
    >
      <circle
        cx="50"
        cy="50"
        r="48"
        fill="none"
        stroke="black"
        stroke-width="0.8"
        stroke-dasharray="8 6"
        vector-effect="non-scaling-stroke"
      />
    </svg>
    <svg
      class="w-12 h-12 sm:w-14 sm:h-14 select-none brightness-0 relative z-10"
      viewBox="0 0 24 24"
      xmlns="http://www.w3.org/2000/svg"
      fill="currentColor"
    >
      <path d="M23 4v2h-3v3h-2V6h-3V4h3V1h2v3h3zm-8.5 7c.828 0 1.5-.672 1.5-1.5S15.328 8 14.5 8 13 8.672 13 9.5s.672 1.5 1.5 1.5zm3.5 3.234l-.513-.57c-.794-.885-2.18-.885-2.976 0l-.655.73L9 9l-3 3.333V6h7V4H6c-1.105 0-2 .895-2 2v12c0 1.105.895 2 2 2h12c1.105 0 2-.895 2-2v-7h-2v3.234z"/>
    </svg>
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
