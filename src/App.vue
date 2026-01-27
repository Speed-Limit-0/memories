<script setup lang="ts">
import Kaleidoscope from './components/Kaleidoscope.vue';
import SegmentedControl, { Option } from './components/SegmentedControl.vue';
import {onUpdated, ref, useTemplateRef} from 'vue';
import SteeringControl from './components/SteeringControl.vue';
import { ScopeShape } from './scopeShape.ts';
import Prompt from './components/Prompt.vue';
import IconButton from './components/IconButton.vue';
import CaptureModal from './components/CaptureModal.vue';

const options: Option[] = [
  {
    buttonAriaLabel: 'Equilateral',
    imageUrl: '/equilateral.svg',
  },
  {
    buttonAriaLabel: 'Square',
    imageUrl: '/square.svg',
  },
];

const showPrompt = ref(true);
const showInfo = ref(false);
const saveNextFrame = ref(false);
const capturedFrame = ref(null as null|string);
const showCapture = ref(false);
const showControls = ref(true);
const selectedIndex = ref(ScopeShape.Equilateral);
const scopeAutoRotationVelocity = ref(0);
const uploadedImage = ref(null as null|string);
const fileInputRef = useTemplateRef('file-input');
function updateSelectedIndex (value: number) {
  selectedIndex.value = value;
}

function updateScopeAutoRotationVelocity(value: number) {
  scopeAutoRotationVelocity.value = value;
}

function dismissPrompt() {
  showPrompt.value = false;
  showInfo.value = false;
}

function savedFrame(objectUrl: string) {
  saveNextFrame.value = false;
  showCapture.value = true;
  capturedFrame.value = objectUrl;
}

const handleFileUpload = (event: Event) => {
  const target = event.target as HTMLInputElement;
  const file = target.files?.[0];
  if (file && file.type.startsWith('image/')) {
    const reader = new FileReader();
    reader.onload = (e) => {
      const result = e.target?.result;
      if (typeof result === 'string') {
        uploadedImage.value = result;
      }
    };
    reader.readAsDataURL(file);
  }
};

const handleUploadClick = () => {
  fileInputRef.value?.click();
};

window.addEventListener('keypress', (keyEvent) => {
  if (keyEvent.code == 'Digit1') {
    selectedIndex.value = ScopeShape.Equilateral;
  } else if (keyEvent.code == 'Digit2') {
    selectedIndex.value = ScopeShape.Square;
  }
  if (keyEvent.code != 'KeyH') {
    return;
  }
  if (showPrompt.value) {
    return;
  }
  showControls.value = !showControls.value;
});

const controlsBarRef = useTemplateRef('controls-bar');

onUpdated(() => {
  const controlsBar = controlsBarRef.value as HTMLDivElement | null;
  if (controlsBar !== null) {
    controlsBar.addEventListener('touchstart', (e) => { e.preventDefault(); });
    controlsBar.addEventListener('touchmove', (e) => { e.preventDefault(); });
  }
});

</script>

<template>
  <Kaleidoscope
    v-if="!showPrompt"
    :scope-shape="selectedIndex"
    :scope-auto-rotation-velocity="scopeAutoRotationVelocity"
    :save-next-frame="saveNextFrame"
    :uploaded-image="uploadedImage"
    @save-frame="savedFrame"
  />
  <div
    v-if="!showPrompt && !showInfo && !showCapture && showControls"
    class="absolute w-full flex flex-col justify-end gap-1 px-2 py-1 items-center pointer-events-none"
    style="bottom:calc(env(safe-area-inset-bottom))"
  >
    <SteeringControl
      class="w-full max-w-56 h-6"
      @update:velocity="updateScopeAutoRotationVelocity"
    />
    <div
      ref="controls-bar"
      class="
      pointer-events-none
      w-full max-w-72 sm:max-w-64 flex flex-row
      items-center
      p-1
      relative
      bg-neutral-300/50
      dark:bg-neutral-800/70
      shadow-lg
      dark:shadow-none
      backdrop-blur-xl
      rounded-xl
      text-sm
      overflow-hidden
      justify-between
      "
    >
      <SegmentedControl
        class="w-full pointer-events-auto basis-4/6"
        :options="options"
        :selected-index="selectedIndex"
        @update:selected-index="updateSelectedIndex"
      />
      <div
        class="flex justify-center basis-[calc(100%_*_1_/_24)]"
      >
        <div
          class="bg-white/30 rounded-full w-0.5 h-6"
        />
      </div>
      <div
        class="flex justify-between pointer-events-auto basis-[calc(100%_*_8_/_24)] gap-1"
      >
        <IconButton
          image-url="/capture.svg"
          label="Capture"
          class="w-full"
          @press="saveNextFrame = true"
        />
        <IconButton
          image-url="/upload.svg"
          label="Upload image"
          class="w-full"
          @press="handleUploadClick"
        />
        <IconButton
          image-url="/info.svg"
          class="w-full"
          label="Info"
          @press="showInfo = true"
        />
      </div>
      <input
        ref="file-input"
        type="file"
        accept="image/*"
        class="hidden"
        @change="handleFileUpload"
      >
    </div>
  </div>
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
  <div
    v-if="showInfo"
    class="absolute left-0 top-0 h-screen w-screen grid grid-cols-1 grid-rows-1 p-2 overflow-auto bg-black/20 backdrop-blur-xl"
    style="width:100dvw;height:100dvh;background-size: cover;"
  >
    <Prompt
      :is-resume-prompt="true"
      @click:enter="dismissPrompt"
    />
  </div>
  <div
    v-if="showPrompt"
    class="absolute left-0 top-0 h-screen w-screen grid grid-cols-1 grid-rows-1 p-2 overflow-auto"
    style="width:100dvw;height:100dvh;background-size: cover;"
    :style="{
      'background-image': `url('/kaleidoscope-${Math.ceil(Math.random() * 10)}.jpg')`
    }"
  >
    <Prompt
      @click:enter="dismissPrompt"
    />
  </div>
</template>

<style scoped>
</style>
