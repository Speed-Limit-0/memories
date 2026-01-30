<script setup lang="ts">
import {onMounted, ref, useTemplateRef} from 'vue';

defineProps<{
  label: string,
  imageUrl: string
}>();

const emit = defineEmits(['press']);

const pressActive = ref(false);
const buttonRef = useTemplateRef('button');

onMounted(() => {
  const button = buttonRef.value as HTMLButtonElement;
  // Mouse events
  let isMouseDown = false;
  button.addEventListener('mousedown', () => {
    isMouseDown = true;
    pressActive.value = true;
  });
  button.addEventListener('mouseleave', () => {
    pressActive.value = false;
  });
  window.addEventListener('mouseup', () => {
    isMouseDown = false;
    pressActive.value = false;
  });
  button.addEventListener('mouseenter', () => {
    if (isMouseDown) {
      pressActive.value = true;
    }
  });

  // Touch events
  button.addEventListener('touchstart', () => {
    pressActive.value = true;
  });
  button.addEventListener('touchmove', (e) => {
    const element = document.elementFromPoint(
        e.changedTouches[0].clientX,
        e.changedTouches[0].clientY,
    );
    pressActive.value = element === button;
  });
  button.addEventListener('touchend', (e) => {
    const element = document.elementFromPoint(
        e.changedTouches[0].clientX,
        e.changedTouches[0].clientY,
    );
    if (element === button) {
      emit('press');
    }
  });
  // Accessibility events
  button.addEventListener('click', () => {
    isMouseDown = false;
    emit('press');
  });
  function reset() {
    pressActive.value = false;
  }
  window.addEventListener('pointerup', reset);
  button.addEventListener('pointercancel', reset);
});
</script>

<template>
  <button
    ref="button"
    class="
      pointer-events-auto
      h-full py-2 rounded-lg
      flex justify-center items-center
      cursor-pointer
    "
    :aria-label="label"
    :title="label"
  >
    <img
      class="
        h-5 w-5 shrink-0
        pointer-events-none
        transition-opacity ease-out duration-300
      "
      :class="{
        'opacity-100 dark:opacity-70': !pressActive,
        'opacity-80 dark:opacity-50': pressActive
      }"
      aria-hidden="true"
      :src="imageUrl"
      alt=""
    >
  </button>
</template>

<style scoped>

</style>
