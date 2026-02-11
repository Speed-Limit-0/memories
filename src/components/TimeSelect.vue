<script setup lang="ts">
import { useTemplateRef, computed } from 'vue';
import RotaryTimePicker from './RotaryTimePicker.vue';

const emit = defineEmits<{
  (e: 'start'): void;
}>();

const rotaryPickerRef = useTemplateRef<InstanceType<typeof RotaryTimePicker>>('rotary-picker');

const isSpinning = computed(() => rotaryPickerRef.value?.spinning ?? false);

const handleRandom = () => {
  if (isSpinning.value) return;
  rotaryPickerRef.value?.randomize();
};
</script>

<template>
  <div class="time-select-screen w-full h-[100dvh] flex flex-col">
    <!-- Header - left-aligned per Figma 1281-456 -->
    <header class="pt-10 px-7 pb-4 text-left">
      <p class="text-sm font-normal text-black mb-1">
        What can you remember?
      </p>
      <h1 class="header-title text-2xl text-black leading-tight">
        Select a time period<br />to test your memory.
      </h1>
    </header>

    <!-- Rotary time picker: oversized, clipped by viewport edges -->
    <div class="dial-viewport-clip">
      <div class="dial-centering">
        <div class="dial-clip-container relative">
          <!-- The dial - full circle, oversized to clip at viewport edges -->
          <div class="dial-wrapper">
            <RotaryTimePicker ref="rotary-picker" />
          </div>
          <!-- Gradient fade at clipped edge -->
          <div class="fade-overlay"></div>
        </div>
      </div>
    </div>

    <!-- Random button: 24px below year dial curve -->
    <div class="random-button-wrap">
      <button
        type="button"
        :class="['random-button', { 'random-button--disabled': isSpinning }]"
        :disabled="isSpinning"
        @click="handleRandom"
      >
        <img src="/random_sort.svg" alt="" class="random-icon" width="24" height="24" />
        <span class="random-label">Random</span>
      </button>
    </div>

    <!-- CTA: Frame 115 - View your orbs + arrow -->
    <div class="cta-wrap">
      <button
        type="button"
        :class="['cta-view-orbs', { 'cta-view-orbs--disabled': isSpinning }]"
        :disabled="isSpinning"
        @click="emit('start')"
      >
        <span class="cta-view-orbs__label">View your orbs</span>
        <span class="cta-view-orbs__icon-wrap">
          <img src="/arrow.svg?v=2" alt="" class="cta-view-orbs__icon" width="32" height="32" />
        </span>
      </button>
    </div>
  </div>
</template>

<style scoped>
.time-select-screen {
  font-family: "Aspekta 250", sans-serif;
  font-weight: 250;
  /* Pin to 16px so design scales 1:1 regardless of app root font-size */
  font-size: 16px;
}

/* Outer container: full viewport width, clips overflow */
.dial-viewport-clip {
  flex: 1;
  width: 100%;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  transform: translateY(-36px);
}

/* Centering wrapper for the oversized dial */
.dial-centering {
  display: flex;
  justify-content: center;
}

.dial-clip-container {
  /* Oversized: 1.5x original scale (scaled down from 2.25x) */
  width: calc(150vw + 144px);
  /* Show only top ~55% of the dial (accounting for extra 60px in viewBox) */
  aspect-ratio: 400 / 253;
  overflow: hidden;
}

.dial-wrapper {
  /* Match SVG viewBox aspect ratio (400 x 460) */
  width: 100%;
  aspect-ratio: 400 / 460;
}

.fade-overlay {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  height: 80px;
  /* Match page background color #EAEAE8 */
  background: linear-gradient(to bottom, transparent, #EAEAE8);
  pointer-events: none;
}

/* Random button */
.random-button-wrap {
  display: flex;
  justify-content: center;
  flex-shrink: 0;
  transform: translateY(-100px);
  position: relative;
  z-index: 5;
}

.random-button {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
  width: 152px;
  height: 74px;
  border-radius: 50%;
  border: 1px solid #292929;
  background: none;
  cursor: pointer;
  -webkit-tap-highlight-color: transparent;
  padding: 0;
  transition: border-color 0.4s ease;
}
.random-button:hover { opacity: 0.85; }
.random-button:active { opacity: 0.7; }
.random-button--disabled {
  border-color: #BCBCBC;
  cursor: default;
  pointer-events: none;
}
.random-button--disabled .random-icon {
  filter: grayscale(1) opacity(0.26);
}
.random-button--disabled .random-label {
  color: #BCBCBC;
}

.random-icon {
  width: 24px;
  height: 24px;
  flex-shrink: 0;
  transition: filter 0.4s ease;
}

.random-label {
  font-family: "Aspekta 250", sans-serif;
  font-weight: 300;
  font-size: 16px;
  letter-spacing: -0.32px;
  color: #292929;
  line-height: normal;
  transition: color 0.4s ease;
}

/* Frame 115: View your orbs CTA */
.cta-wrap {
  padding: 0 28px 40px 28px;
  display: flex;
  justify-content: flex-start;
  flex-shrink: 0;
}
.cta-view-orbs {
  display: flex;
  flex-direction: row;
  align-items: center;
  padding: 0;
  gap: 12px;
  width: 247px;
  height: 46px;
  background: none;
  border: none;
  cursor: pointer;
  -webkit-tap-highlight-color: transparent;
}
.cta-view-orbs {
  transition: opacity 0.4s ease;
}
.cta-view-orbs:hover { opacity: 0.85; }
.cta-view-orbs:active { opacity: 0.7; }
.cta-view-orbs--disabled {
  opacity: 0;
  pointer-events: none;
}
.header-title { font-weight: 200; }
.cta-view-orbs__label {
  font-family: inherit;
  font-size: 1.5rem;   /* text-2xl, same as h1 */
  font-weight: 200;
  line-height: 1.25;   /* leading-tight */
  text-align: center;
  color: #000;
  flex: none;
  order: 0;
  flex-grow: 0;
}
.cta-view-orbs__icon-wrap {
  width: 32px;
  height: 32px;
  flex: none;
  order: 1;
  flex-grow: 0;
  display: flex;
  align-items: center;
  justify-content: center;
}
.cta-view-orbs__icon {
  display: block;
  animation: subtle-bounce 1.4s ease-in-out infinite;
}

@keyframes subtle-bounce {
  0%, 100% { transform: translateY(-2px); }
  50% { transform: translateY(2px); }
}
</style>
