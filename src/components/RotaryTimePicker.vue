<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref, useTemplateRef, readonly } from 'vue';

const MONTHS = [
  'Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun',
  'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec',
];
const YEARS = [2020, 2021, 2022, 2023, 2024, 2025, 2026, 2027, 2028, 2029, 2030];

const MONTH_STEP = 360 / 12;
const YEAR_STEP = 360 / 11;

const emit = defineEmits<{
  (e: 'month', value: number): void;
  (e: 'year', value: number): void;
}>();

const containerRef = useTemplateRef<HTMLDivElement>('container');
// Labels are now at segment centers: index i is at (i + 0.5) * step degrees
// To put February (index 1) at top: rotate by -(1 + 0.5) * step = -1.5 * step
const monthRotation = ref(-1.5 * MONTH_STEP); // February at top
// To put 2024 (index 4) at top: rotate by -(4 + 0.5) * step = -4.5 * step
const yearRotation = ref(-4.5 * YEAR_STEP);   // 2024 at top
const activeDial = ref<'month' | 'year' | null>(null);
const startAngle = ref(0);
const startRotation = ref(0);
const spinning = ref(false);

const selectedMonthIndex = computed(() => {
  // Labels at segment centers: index i at (i + 0.5) * step
  // Selected index from rotation: (rotation / -step) - 0.5
  const idx = Math.round(-monthRotation.value / MONTH_STEP - 0.5);
  return ((idx % 12) + 12) % 12;
});
const selectedYearIndex = computed(() => {
  const idx = Math.round(-yearRotation.value / YEAR_STEP - 0.5);
  return ((idx % 11) + 11) % 11;
});

const size = 400;
const cx = size / 2;
const cy = size / 2;

// Outer dial: months - text on outer circle, dotted track inside
const monthTextRadius = 160;    // text path radius (outermost)
const monthTrackRadius = 150;   // dotted circle below text

// Inner dial: years - text on inner circle, dotted track inside
const yearTextRadius = 100;     // text path radius
const yearTrackRadius = 90;     // dotted circle below text

const innerRingRadius = 105;    // boundary: inside = year, outside = month

function mod(n: number, d: number) {
  return ((n % d) + d) % d;
}

function angleFromPoint(rect: DOMRect, clientX: number, clientY: number): number {
  const x = clientX - rect.left - rect.width / 2;
  const y = clientY - rect.top - rect.height / 2;
  return (Math.atan2(y, x) * 180) / Math.PI;
}

function distFromCenter(rect: DOMRect, clientX: number, clientY: number): number {
  const x = clientX - rect.left - rect.width / 2;
  const y = clientY - rect.top - rect.height / 2;
  return Math.sqrt(x * x + y * y);
}

function hitTest(rect: DOMRect, clientX: number, clientY: number): 'month' | 'year' | null {
  const d = distFromCenter(rect, clientX, clientY);
  const scale = rect.width / size;
  const dScaled = d / scale;
  if (dScaled < innerRingRadius) return 'year';
  if (dScaled <= monthTextRadius + 30) return 'month';
  return null;
}

function pointerDown(e: PointerEvent) {
  // Block interaction while spinning
  if (spinning.value) return;

  // Cancel any in-progress snap animation
  if (snapAnimId !== null) {
    cancelAnimationFrame(snapAnimId);
    snapAnimId = null;
  }

  const el = containerRef.value;
  if (!el) return;
  const rect = el.getBoundingClientRect();
  const dial = hitTest(rect, e.clientX, e.clientY);
  if (!dial) return;

  e.preventDefault();
  el.setPointerCapture(e.pointerId);
  activeDial.value = dial;
  startAngle.value = angleFromPoint(rect, e.clientX, e.clientY);
  startRotation.value = dial === 'month' ? monthRotation.value : yearRotation.value;
}

function pointerMove(e: PointerEvent) {
  if (activeDial.value === null) return;
  e.preventDefault();
  const el = containerRef.value;
  if (!el) return;
  const rect = el.getBoundingClientRect();
  const currentAngle = angleFromPoint(rect, e.clientX, e.clientY);
  const delta = currentAngle - startAngle.value;
  startAngle.value = currentAngle;

  if (activeDial.value === 'month') {
    monthRotation.value = startRotation.value + delta;
    startRotation.value = monthRotation.value;
  } else {
    yearRotation.value = startRotation.value + delta;
    startRotation.value = yearRotation.value;
  }
}

// --- Smooth snap animation ---
const SNAP_DURATION = 320; // ms
let snapAnimId: number | null = null;

// Ease-out cubic: fast start, smooth deceleration
function easeOutCubic(t: number): number {
  return 1 - Math.pow(1 - t, 3);
}

function animateSnap(
  rotationRef: typeof monthRotation,
  fromAngle: number,
  toAngle: number,
  onDone: () => void,
) {
  if (snapAnimId !== null) cancelAnimationFrame(snapAnimId);
  const start = performance.now();

  function tick(now: number) {
    const elapsed = now - start;
    const t = Math.min(elapsed / SNAP_DURATION, 1);
    const eased = easeOutCubic(t);
    rotationRef.value = fromAngle + (toAngle - fromAngle) * eased;

    if (t < 1) {
      snapAnimId = requestAnimationFrame(tick);
    } else {
      snapAnimId = null;
      onDone();
    }
  }

  snapAnimId = requestAnimationFrame(tick);
}

function pointerUp() {
  if (activeDial.value === 'month') {
    const idx = mod(Math.round(-monthRotation.value / MONTH_STEP - 0.5), 12);
    const target = -(idx + 0.5) * MONTH_STEP;
    const from = monthRotation.value;
    activeDial.value = null;
    animateSnap(monthRotation, from, target, () => emit('month', idx));
  } else if (activeDial.value === 'year') {
    const idx = mod(Math.round(-yearRotation.value / YEAR_STEP - 0.5), 11);
    const target = -(idx + 0.5) * YEAR_STEP;
    const from = yearRotation.value;
    activeDial.value = null;
    animateSnap(yearRotation, from, target, () => emit('year', idx));
  } else {
    activeDial.value = null;
  }
}

// --- Slot-machine style spin ---
const SPIN_DURATION = 2400; // ms total
const SPIN_EXTRA_ROTATIONS_MONTH = 4; // full rotations before landing
const SPIN_EXTRA_ROTATIONS_YEAR = 3;

// Ease-out quint for dramatic deceleration (fast start, slow finish)
function easeOutQuint(t: number): number {
  return 1 - Math.pow(1 - t, 5);
}

let spinAnimId: number | null = null;

function randomize() {
  // Cancel any in-progress animations
  if (snapAnimId !== null) { cancelAnimationFrame(snapAnimId); snapAnimId = null; }
  if (spinAnimId !== null) { cancelAnimationFrame(spinAnimId); spinAnimId = null; }

  spinning.value = true;

  // Pick random targets (ensure at least one changes)
  let randomMonthIdx: number;
  let randomYearIdx: number;
  do {
    randomMonthIdx = Math.floor(Math.random() * 12);
    randomYearIdx = Math.floor(Math.random() * 11);
  } while (randomMonthIdx === selectedMonthIndex.value && randomYearIdx === selectedYearIndex.value);

  // Target snap angles
  const monthSnap = -(randomMonthIdx + 0.5) * MONTH_STEP;
  const yearSnap = -(randomYearIdx + 0.5) * YEAR_STEP;

  // Current positions
  const monthFrom = monthRotation.value;
  const yearFrom = yearRotation.value;

  // Total travel: multiple full rotations + shortest path to target
  // Always spin in the same direction (negative = clockwise visually)
  const monthTotal = -(SPIN_EXTRA_ROTATIONS_MONTH * 360) + (monthSnap - monthFrom);
  const yearTotal = -(SPIN_EXTRA_ROTATIONS_YEAR * 360) + (yearSnap - yearFrom);

  const start = performance.now();

  function tick(now: number) {
    const elapsed = now - start;
    const t = Math.min(elapsed / SPIN_DURATION, 1);
    const eased = easeOutQuint(t);

    monthRotation.value = monthFrom + monthTotal * eased;
    yearRotation.value = yearFrom + yearTotal * eased;

    if (t < 1) {
      spinAnimId = requestAnimationFrame(tick);
    } else {
      // Snap exactly to target
      monthRotation.value = monthSnap;
      yearRotation.value = yearSnap;
      spinAnimId = null;
      spinning.value = false;
      emit('month', randomMonthIdx);
      emit('year', randomYearIdx);
    }
  }

  spinAnimId = requestAnimationFrame(tick);
}

defineExpose({ randomize, spinning: readonly(spinning) });

// Create an arc path for a single label segment
// startAngle and endAngle in degrees, 0° = top, clockwise
function createArcPath(r: number, startAngleDeg: number, endAngleDeg: number): string {
  // Convert to radians, offset by -90° so 0° is at top
  const startRad = ((startAngleDeg - 90) * Math.PI) / 180;
  const endRad = ((endAngleDeg - 90) * Math.PI) / 180;
  
  const x1 = cx + r * Math.cos(startRad);
  const y1 = cy + r * Math.sin(startRad);
  const x2 = cx + r * Math.cos(endRad);
  const y2 = cy + r * Math.sin(endRad);
  
  // Large arc flag: 0 for arcs < 180°
  const largeArc = (endAngleDeg - startAngleDeg) > 180 ? 1 : 0;
  
  return `M ${x1} ${y1} A ${r} ${r} 0 ${largeArc} 1 ${x2} ${y2}`;
}

// Generate arc paths for each month (12 segments of 30° each)
function getMonthArcPath(i: number): string {
  const segmentAngle = 360 / 12;
  const startAngle = i * segmentAngle;
  const endAngle = (i + 1) * segmentAngle;
  return createArcPath(monthTextRadius, startAngle, endAngle);
}

// Generate arc paths for each year (11 segments of ~32.7° each)
function getYearArcPath(i: number): string {
  const segmentAngle = 360 / 11;
  const startAngle = i * segmentAngle;
  const endAngle = (i + 1) * segmentAngle;
  return createArcPath(yearTextRadius, startAngle, endAngle);
}

onMounted(() => {
  window.addEventListener('pointermove', pointerMove, { passive: false });
  window.addEventListener('pointerup', pointerUp);
  window.addEventListener('pointercancel', pointerUp);
});
onUnmounted(() => {
  window.removeEventListener('pointermove', pointerMove);
  window.removeEventListener('pointerup', pointerUp);
  window.removeEventListener('pointercancel', pointerUp);
});
</script>

<template>
  <div
    ref="container"
    class="rotary-picker touch-none select-none cursor-grab active:cursor-grabbing"
    @pointerdown="pointerDown"
  >
    <svg
      :viewBox="`0 -60 ${size} ${size + 60}`"
      class="w-full h-full max-w-full max-h-full mx-auto block"
      aria-hidden="true"
    >
      <!-- Selection indicator at top (17px line) -->
      <line
        :x1="cx"
        :y1="cy - monthTextRadius - 49"
        :x2="cx"
        :y2="cy - monthTextRadius - 32"
        stroke="#000"
        stroke-width="0.5"
        stroke-linecap="round"
      />
      <!-- Year dial selection line: height 14, gap above year dial 24 -->
      <line
        :x1="cx"
        :y1="cy - yearTextRadius - 24 - 14"
        :x2="cx"
        :y2="cy - yearTextRadius - 24"
        stroke="#000"
        stroke-width="0.5"
        stroke-linecap="round"
      />

      <!-- Outer dial: months -->
      <g :transform="`rotate(${monthRotation}, ${cx}, ${cy})`">
        <!-- Dotted track (inside text, smaller radius) -->
        <circle
          :cx="cx"
          :cy="cy"
          :r="monthTrackRadius"
          fill="none"
          stroke="#000"
          stroke-width="0.75"
          stroke-dasharray="0.1 8"
          stroke-linecap="round"
        />
        <!-- Each month has its own arc path for natural text centering -->
        <template v-for="(name, i) in MONTHS" :key="name">
          <path
            :id="`monthPath${i}`"
            :d="getMonthArcPath(i)"
            fill="none"
            stroke="none"
          />
          <text
            :class="[
              'month-label',
              i === selectedMonthIndex ? 'selected' : 'unselected'
            ]"
          >
            <textPath
              :href="`#monthPath${i}`"
              startOffset="50%"
              text-anchor="middle"
            >
              {{ name }}
            </textPath>
          </text>
        </template>
      </g>

      <!-- Inner dial: years -->
      <g :transform="`rotate(${yearRotation}, ${cx}, ${cy})`">
        <!-- Dotted track (inside text, smaller radius) -->
        <circle
          :cx="cx"
          :cy="cy"
          :r="yearTrackRadius"
          fill="none"
          stroke="#000"
          stroke-width="0.75"
          stroke-dasharray="0.1 8"
          stroke-linecap="round"
        />
        <!-- Each year has its own arc path for natural text centering -->
        <template v-for="(yr, i) in YEARS" :key="yr">
          <path
            :id="`yearPath${i}`"
            :d="getYearArcPath(i)"
            fill="none"
            stroke="none"
          />
          <text
            :class="[
              'year-label',
              i === selectedYearIndex ? 'selected' : 'unselected'
            ]"
          >
            <textPath
              :href="`#yearPath${i}`"
              startOffset="50%"
              text-anchor="middle"
            >
              {{ yr }}
            </textPath>
          </text>
        </template>
      </g>
    </svg>
  </div>
</template>

<style scoped>
.month-label {
  font-family: "PP Neue Montreal", sans-serif;
  font-weight: 100;
  font-size: 22px;
  fill: #000;
}
.month-label.selected {
  font-weight: 300;
  font-size: 26px;
}
.month-label.unselected {
  font-weight: 100;
  opacity: 0.5;
}
.year-label {
  font-family: "PP Neue Montreal", sans-serif;
  font-weight: 100;
  font-size: 14px;
  fill: #000;
}
.year-label.selected {
  font-weight: 300;
  font-size: 18px;
}
.year-label.unselected {
  font-weight: 100;
  opacity: 0.5;
}
</style>
