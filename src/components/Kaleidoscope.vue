<script setup lang="ts">
import {ref, onMounted, nextTick, useTemplateRef, watch, onUnmounted} from 'vue';
import { ScopeShape } from '../scopeShape.ts';

const props = defineProps<{
  scopeShape: ScopeShape,
  scopeAutoRotationVelocity: number
  saveNextFrame: boolean
  uploadedImages?: string[]
}>();

const emit = defineEmits(['save-frame', 'upload-click', 'remove-uploaded-image']);

const CLICK_MOVE_THRESHOLD_PX = 10;

const facingMode = ref('unknown');
const cameraZoom = ref(1);
const scopeRotation = ref(0.0);
// Independent rotations for each orb
const scopeRotationTop = ref(0.0);
const scopeRotationBottom = ref(0.0);
const scopeSize = ref(1);
const scopeRotationTopVel = ref(0.0);
const scopeRotationBottomVel = ref(0.0);
const scopeOffset = ref([0.0, 0.0]);
const scopeOffsetVel = ref([0.0, 0.0]);
const scopeSizeVel = ref(0.0);
const scopeRotationVel = ref(0.0);
const isUserPressing = ref(false);
const keyPressedShift = ref(false);
const keyPressedAlt = ref(false);
const keyPressedW = ref(false);
const keyPressedA = ref(false);
const keyPressedS = ref(false);
const keyPressedD = ref(false);
const keyPressedI = ref(false);
const keyPressedJ = ref(false);
const keyPressedK = ref(false);
const keyPressedL = ref(false);
const keyPressedMinus = ref(false);
const keyPressedPlus = ref(false);
const canvas = useTemplateRef('canvas');
const displayCanvasCenter = useTemplateRef('display-canvas-center');
const displayCanvasTop = useTemplateRef('display-canvas-top');
const displayCanvasBottom = useTemplateRef('display-canvas-bottom');
const displayCanvasIncoming = useTemplateRef('display-canvas-incoming');
const interactionLayer = useTemplateRef('interaction-layer');
const centerOrb = useTemplateRef('center-orb');
const uploadedImageElements = ref([] as HTMLImageElement[]);
const activeImageIndex = ref(0);
const topSlotIndex = ref(0);
const centerSlotIndex = ref(0);
const bottomSlotIndex = ref(0);
const orbTransitionProgress = ref(0.0);
const isOrbTransitioning = ref(false);
const orbTransitionDirection = ref<1 | -1>(1);
const orbScrollOffset = ref(0);
const orbScrollVel = ref(0);
// Simplified impulse mapping (no separate multipliers)
// Simplified impulse mapping parameters (single mapping for wheel/trackpad/touch)
const ORB_WHEEL_DELTA_MAX = 120; // clamp reference for raw wheel delta
const ORB_MAX_IMPULSE = 3.0; // max impulse (progress units) for strongest flick
const ORB_IMPULSE_EXP = 1.2; // nonlinear exponent (>1 makes large deltas grow faster)
const ORB_TRACKPAD_SCALE = 0.9; // slight device scale for trackpad
const ORB_FRICTION = 10; // lower = less friction (was 8)
const ORB_VELOCITY_THRESHOLD = 1.2;
const ORB_MAX_VELOCITY = 20;
const maxRotationSpeed = 1; // Maximum rotation velocity
const maxScopeSizeVel = 0.12; // Maximum zoom velocity for physics follow-through
let cameraStream: MediaStream | null = null;
// Scroll -> rotation mapping: scale factor applied to scroll velocity (px/ms) to rotation velocity
const SCROLL_ROTATION_SCALE = 0.02;
// Track last window scroll position/time to compute scroll velocity
let lastWindowScrollY = 0;
let lastWindowScrollTime = performance.now();

// Horizontal drag-to-dismiss (removes the active uploaded image)
const ORB_DISMISS_DURATION_MS = 200;
const orbDragTranslateX = ref(0);
const orbDragTranslateY = ref(0);
const orbDragScaleMultiplier = ref(1);
const isOrbDragging = ref(false);
const isOrbDismissing = ref(false);
const isCenterHiddenDuringDismiss = ref(false);
const orbFillAnimating = ref(false);
let orbDragMouseStart: null | { x: number; y: number } = null;
let orbDragMouseActive = false;
let orbDragTouchActive = false;
let orbDragTouchLast: null | Point = null;
let orbDragLastX: null | number = null;
const pendingOrbRemovalIndex = ref<number | null>(null);

const clampRotationVelocity = (velocity: number): number => {
  return Math.max(-maxRotationSpeed, Math.min(maxRotationSpeed, velocity));
};
const clampScopeSizeVelocity = (velocity: number): number => {
  return Math.max(-maxScopeSizeVel, Math.min(maxScopeSizeVel, velocity));
};
let texture1: WebGLTexture | null = null;
let texture2: WebGLTexture | null = null;

// Maximum texture dimension to reduce upload time and prevent stuttering
const MAX_TEXTURE_SIZE = 2048;

// Resize image to maximum size while maintaining aspect ratio
const resizeImage = (img: HTMLImageElement): Promise<HTMLImageElement> => {
  return new Promise((resolve) => {
    const maxDim = Math.max(img.width, img.height);
    if (maxDim <= MAX_TEXTURE_SIZE) {
      // Image is already small enough
      resolve(img);
      return;
    }
    
    // Calculate new dimensions maintaining aspect ratio
    const scale = MAX_TEXTURE_SIZE / maxDim;
    const newWidth = Math.round(img.width * scale);
    const newHeight = Math.round(img.height * scale);
    
    // Create canvas to resize image
    const canvas = document.createElement('canvas');
    canvas.width = newWidth;
    canvas.height = newHeight;
    const ctx = canvas.getContext('2d')!;
    ctx.drawImage(img, 0, 0, newWidth, newHeight);
    
    // Create new image from resized canvas
    const resizedImg = new Image();
    resizedImg.crossOrigin = 'anonymous';
    resizedImg.onload = () => resolve(resizedImg);
    resizedImg.onerror = () => resolve(img); // Fallback to original if resize fails
    resizedImg.src = canvas.toDataURL('image/jpeg', 0.92);
  });
};

const getWrappedIndex = (index: number, length: number): number => {
  if (length <= 0) {
    return 0;
  }
  return ((index % length) + length) % length;
};

const syncSlotIndices = () => {
  const totalImages = uploadedImageElements.value.length;
  if (totalImages <= 0) {
    topSlotIndex.value = 0;
    centerSlotIndex.value = 0;
    bottomSlotIndex.value = 0;
    return;
  }
  centerSlotIndex.value = getWrappedIndex(activeImageIndex.value, totalImages);
  topSlotIndex.value = getWrappedIndex(activeImageIndex.value - 1, totalImages);
  bottomSlotIndex.value = getWrappedIndex(activeImageIndex.value + 1, totalImages);
};

// Apply an impulse to scroll velocity (delta is in "progress" units)
const applyOrbScrollDelta = (delta: number) => {
  const totalImages = uploadedImageElements.value.length;
  if (totalImages <= 1) {
    return;
  }
  // Cancel any active snap while user is providing input
  cancelOrbSnap();
  // delta here is expected to already be a computed impulse value
  orbScrollVel.value += delta;
  // Clamp velocity to avoid runaway
  orbScrollVel.value = Math.max(-ORB_MAX_VELOCITY, Math.min(ORB_MAX_VELOCITY, orbScrollVel.value));
  // Ensure visuals update
  orbTransitionDirection.value = orbScrollOffset.value >= 0 ? 1 : -1;
  orbTransitionProgress.value = Math.min(1, Math.abs(orbScrollOffset.value));
  isOrbTransitioning.value = orbTransitionProgress.value > 0.001;
};

// Convert raw input delta (wheel delta or touch delta) into a sensible impulse.
const computeImpulseFromDelta = (rawDelta: number, isTrackpad: boolean) => {
  const sign = Math.sign(rawDelta) || 1;
  const absClamped = Math.min(ORB_WHEEL_DELTA_MAX, Math.abs(rawDelta));
  const normalized = absClamped / ORB_WHEEL_DELTA_MAX; // 0..1
  const scaled = Math.pow(normalized, ORB_IMPULSE_EXP);
  const base = scaled * ORB_MAX_IMPULSE;
  const deviceScale = isTrackpad ? ORB_TRACKPAD_SCALE : 1;
  return sign * base * deviceScale;
};

// Scroll snapping: animate orbScrollOffset to nearest integer when user stops scrolling
let orbSnapTimeout: number | null = null;
let orbSnapRaf: number | null = null;
let orbLastWheelDelta = 0;
const cancelOrbSnap = () => {
  if (orbSnapTimeout !== null) {
    clearTimeout(orbSnapTimeout);
    orbSnapTimeout = null;
  }
  if (orbSnapRaf !== null) {
    cancelAnimationFrame(orbSnapRaf);
    orbSnapRaf = null;
  }
};
const ORB_FAST_SCROLL_THRESHOLD = 1.0; // progress delta threshold considered a "fast" flick
const startOrbSnap = () => {
  cancelOrbSnap();

  // Normalize orbScrollOffset into (-1,1) by applying any full-step offsets immediately.
  const totalImages = uploadedImageElements.value.length;
  while (orbScrollOffset.value >= 1) {
    if (totalImages > 0) {
      activeImageIndex.value = getWrappedIndex(activeImageIndex.value + 1, totalImages);
    }
    orbScrollOffset.value -= 1;
  }
  while (orbScrollOffset.value <= -1) {
    if (totalImages > 0) {
      activeImageIndex.value = getWrappedIndex(activeImageIndex.value - 1, totalImages);
    }
    orbScrollOffset.value += 1;
  }
  syncSlotIndices();

  const start = orbScrollOffset.value;
  // If the last wheel flick was a fast downward flick, snap to center (0)
  const shouldSnapToCenter = orbLastWheelDelta < -ORB_FAST_SCROLL_THRESHOLD;
  const target = shouldSnapToCenter ? 0 : Math.round(start);
  if (start === target) {
    // No animation needed
    orbTransitionDirection.value = orbScrollOffset.value >= 0 ? 1 : -1;
    orbTransitionProgress.value = Math.min(1, Math.abs(orbScrollOffset.value));
    isOrbTransitioning.value = orbTransitionProgress.value > 0.001;
    // Ensure the center orb inherits rotation from the orb that is moving into center
    const settleDirection = start >= 0 ? 1 : -1;
    if (settleDirection === 1) {
      // Scrolling "up" -> bottom orb becomes center
      scopeRotation.value = scopeRotationBottom.value;
      scopeRotationVel.value = scopeRotationBottomVel.value;
    } else {
      // Scrolling "down" -> top orb becomes center
      scopeRotation.value = scopeRotationTop.value;
      scopeRotationVel.value = scopeRotationTopVel.value;
    }
    return;
  }
  // Spring parameters (tweakable)
  const stiffness = 160; // spring stiffness (higher = stiffer)
  const damping = 50; // damping coefficient (higher = more damped / less bouncy)

  let pos = start;
  let vel = 0;
  let lastTime = performance.now();

  const step = (now: number) => {
    const dt = Math.min(0.032, (now - lastTime) / 1000); // cap dt for stability
    lastTime = now;

    // Hooke's law + damping: a = -k * x - c * v
    const x = pos - target;
    const acc = -stiffness * x - damping * vel;
    vel += acc * dt;
    pos += vel * dt;

    orbScrollOffset.value = pos;
    orbTransitionDirection.value = orbScrollOffset.value >= 0 ? 1 : -1;
    orbTransitionProgress.value = Math.min(1, Math.abs(orbScrollOffset.value));
    isOrbTransitioning.value = orbTransitionProgress.value > 0.001;

    // Stop condition: close to target and very low velocity
    if (Math.abs(pos - target) < 0.002 && Math.abs(vel) < 0.002) {
      // Snap to exact target and finalize index wrap
      orbScrollOffset.value = target;
      const total = uploadedImageElements.value.length;
      while (orbScrollOffset.value >= 1) {
        activeImageIndex.value = getWrappedIndex(activeImageIndex.value + 1, total);
        orbScrollOffset.value -= 1;
      }
      while (orbScrollOffset.value <= -1) {
        activeImageIndex.value = getWrappedIndex(activeImageIndex.value - 1, total);
        orbScrollOffset.value += 1;
      }
      // Transfer rotation from the orb that moved into center for visual continuity
      const settleDirection = start >= 0 ? 1 : -1;
      if (settleDirection === 1) {
        // Scrolling "up" -> bottom orb becomes center
        scopeRotation.value = scopeRotationBottom.value;
        scopeRotationVel.value = scopeRotationBottomVel.value;
      } else {
        // Scrolling "down" -> top orb becomes center
        scopeRotation.value = scopeRotationTop.value;
        scopeRotationVel.value = scopeRotationTopVel.value;
      }
      orbTransitionDirection.value = 1;
      orbTransitionProgress.value = 0;
      isOrbTransitioning.value = false;
      syncSlotIndices();
      orbSnapRaf = null;
      return;
    }

    orbSnapRaf = requestAnimationFrame(step);
  };

  orbSnapRaf = requestAnimationFrame(step);
};

const ORB_SMALL_SCALE = 0.5;
const ORB_SCOPE_SCALE_MULTIPLIER = 0.5;
const ORB_TOP_Y = 0;
const ORB_CENTER_Y = 50;
const ORB_BOTTOM_Y = 100;
const ORB_OFFSCREEN_TOP = -50;
const ORB_OFFSCREEN_BOTTOM = 150;

const lerp = (start: number, end: number, progress: number): number => {
  return start + (end - start) * progress;
};

// Compute orb visual and shader scope scale so the "snapped" orb grows up to 2x.
// Weight mapping: center has weight 1 when fully centered; during transitions the
// weight shifts to the incoming/target slot according to orbScrollOffset progress.
const computeOrbWeight = (slot: 'top' | 'center' | 'bottom' | 'incoming') => {
  const progress = Math.min(1, Math.abs(orbScrollOffset.value));
  const direction = orbScrollOffset.value >= 0 ? 1 : -1;

  if (slot === 'center') {
    return 1 - progress;
  }
  if (slot === 'bottom') {
    return direction === 1 ? progress : 0;
  }
  if (slot === 'top') {
    return direction === -1 ? progress : 0;
  }
  // incoming is an offscreen helper orb and should not scale as the snapped orb.
  if (slot === 'incoming') {
    return 0;
  }
  return 0;
};

const getOrbSizeScale = (slot: 'top' | 'center' | 'bottom' | 'incoming') => {
  const weight = computeOrbWeight(slot);
  // Base visual size multiplied by (1 + weight). weight=1 -> 2x, weight=0 -> 1x.
  return ORB_SMALL_SCALE * (1 + weight);
};

const getOrbScopeScale = (slot: 'top' | 'center' | 'bottom' | 'incoming') => {
  const weight = computeOrbWeight(slot);
  // Animate shader scope size in sync with visual orb scaling.
  return ORB_SCOPE_SCALE_MULTIPLIER * (1 + weight);
};

const getOrbStyle = (slot: 'top' | 'center' | 'bottom' | 'incoming') => {
  const progress = Math.min(1, Math.abs(orbScrollOffset.value));
  const direction = orbScrollOffset.value >= 0 ? 1 : -1;

  let topPercent = slot === 'top' ? ORB_TOP_Y : slot === 'center' ? ORB_CENTER_Y : ORB_BOTTOM_Y;
  if (slot === 'incoming') {
    // default offscreen position based on scroll direction
    topPercent = direction === 1 ? ORB_OFFSCREEN_BOTTOM : ORB_OFFSCREEN_TOP;
  }
  const scale = getOrbSizeScale(slot);

  if (progress > 0) {
    if (direction === 1) {
      // scrolling "up": items move upward. incoming comes from offscreen bottom -> moves to bottom slot
      if (slot === 'top') {
        topPercent = lerp(ORB_TOP_Y, ORB_OFFSCREEN_TOP, progress);
      } else if (slot === 'center') {
        topPercent = lerp(ORB_CENTER_Y, ORB_TOP_Y, progress);
      } else if (slot === 'bottom') {
        topPercent = lerp(ORB_BOTTOM_Y, ORB_CENTER_Y, progress);
      } else if (slot === 'incoming') {
        topPercent = lerp(ORB_OFFSCREEN_BOTTOM, ORB_BOTTOM_Y, progress);
      }
    } else {
      // scrolling "down": items move downward. incoming comes from offscreen top -> moves to top slot
      if (slot === 'bottom') {
        topPercent = lerp(ORB_BOTTOM_Y, ORB_OFFSCREEN_BOTTOM, progress);
      } else if (slot === 'center') {
        topPercent = lerp(ORB_CENTER_Y, ORB_BOTTOM_Y, progress);
      } else if (slot === 'top') {
        topPercent = lerp(ORB_TOP_Y, ORB_CENTER_Y, progress);
      } else if (slot === 'incoming') {
        topPercent = lerp(ORB_OFFSCREEN_TOP, ORB_TOP_Y, progress);
      }
    }
  }

  const extraTranslateX = slot === 'center' ? orbDragTranslateX.value : 0;
  const extraTranslateY = slot === 'center' ? orbDragTranslateY.value : 0;
  // Only the center orb shrinks during drag/dismiss; top/bottom remain fixed
  const extraScale = slot === 'center' ? orbDragScaleMultiplier.value : 1;

  return {
    top: `${topPercent}%`,
    transform: `translate(-50%, -50%) translate(${extraTranslateX}px, ${extraTranslateY}px) scale(${scale * extraScale})`,
  };
};

// Center orb style wrapper so we can hide it during dismiss animations
const getCenterOrbStyle = (): Record<string, string> => {
  const s = getOrbStyle('center') as Record<string, string>;
  if (isCenterHiddenDuringDismiss.value) {
    s.opacity = '0';
    s.pointerEvents = 'none';
  }
  return s;
};

const SWIPE_DISTANCE_PX = 60;
const isSwipeGesture = (deltaX: number, deltaY: number): boolean => {
  return Math.abs(deltaY) >= SWIPE_DISTANCE_PX && Math.abs(deltaY) > Math.abs(deltaX);
};

const handleSwipeGesture = (deltaX: number, deltaY: number): boolean => {
  if (!isSwipeGesture(deltaX, deltaY)) {
    return false;
  }
  // Map swipe distance to impulse (treat as touch, not trackpad)
  const progressDelta = computeImpulseFromDelta(-deltaY, false);
  applyOrbScrollDelta(progressDelta);
  return true;
};

const getReadySource = (
  primary: HTMLImageElement | HTMLVideoElement | null,
  fallback: HTMLImageElement | HTMLVideoElement | null
) => {
  if (primary instanceof HTMLImageElement) {
    if (primary.complete && primary.naturalWidth > 0) {
      return primary;
    }
    return fallback;
  }
  if (primary instanceof HTMLVideoElement) {
    if (primary.readyState >= 2 && primary.videoWidth > 0) {
      return primary;
    }
    return fallback;
  }
  return fallback;
};

const loadUploadedImages = async (imageSrcs: string[]) => {
  if (imageSrcs.length > 0) {
    const loadedImages: HTMLImageElement[] = [];
    for (const src of imageSrcs) {
      const img = new Image();
      img.crossOrigin = 'anonymous';
      img.src = src;
      await new Promise((resolve, reject) => {
        img.onload = resolve;
        img.onerror = reject;
      });
      // Resize image to reduce upload time and prevent stuttering
      const resizedImg = await resizeImage(img);
      loadedImages.push(resizedImg);
    }
    uploadedImageElements.value = loadedImages;
    activeImageIndex.value = 0;
    orbScrollOffset.value = 0;
    syncSlotIndices();
    orbTransitionProgress.value = 0.0;
    isOrbTransitioning.value = false;
    facingMode.value = 'environment'; // Don't flip uploaded images (x-flip is only for user-facing webcam)
    
    // Stop camera stream if images are uploaded
    if (cameraStream) {
      cameraStream.getTracks().forEach(track => track.stop());
      cameraStream = null;
    }
  } else {
    uploadedImageElements.value = [];
    activeImageIndex.value = 0;
    orbScrollOffset.value = 0;
    syncSlotIndices();
    orbTransitionProgress.value = 0.0;
    isOrbTransitioning.value = false;
  }
};

async function main(canvasElement: HTMLCanvasElement) {
  // Capture webcam input using invisible `video` element
  // Adapted from p5js.org/examples/3d-shader-using-webcam.html
  const camera = document.getElementById('camera') as HTMLVideoElement;

  // If images are uploaded, create image elements for them
  if (props.uploadedImages && props.uploadedImages.length > 0) {
    await loadUploadedImages(props.uploadedImages);
  } else {
    // Ask user permission to record their camera
    try {
      cameraStream = await navigator.mediaDevices.getUserMedia({video: { facingMode: { exact: 'environment'} }, audio: false});
      facingMode.value = cameraStream.getVideoTracks()[0]?.getSettings().facingMode ?? 'user';
    } catch (e) {
      console.info('Failed to get environment camera. Trying any camera, under the assumption it is a user-facing camera', e);
      try {
        cameraStream = await navigator.mediaDevices.getUserMedia({video: true, audio: false});
        facingMode.value = cameraStream.getVideoTracks()[0]?.getSettings().facingMode ?? 'user';
      } catch (e2) {
        if ((e2 as Error).name === 'ConstraintNotSatisfiedError') {
          console.error('Device has no camera', e2);
        } else if ((e2 as Error).name === 'PermissionDeniedError') {
          console.error('Permissions not accepted', e2);
        } else {
          console.error('Other error', e2);
        }
      }
    }

    if (cameraStream !== null) {
      camera.srcObject = cameraStream;
      camera.play();
    }
  }

  // Canvas with WebGL context (element passed from template ref so it exists when mounted)
  const canvasSize = Math.max(1024, window.innerWidth, window.innerHeight) * window.devicePixelRatio;
  const gl = canvasElement.getContext('webgl')!;
  canvasElement.width = canvasElement.height = canvasSize;
  gl.viewport(0, 0, canvasElement.width, canvasElement.height);

  // Vertex shader: Identity map
  const vshader = gl.createShader(gl.VERTEX_SHADER)!;
  gl.shaderSource(vshader,
      'attribute vec2 p;'+
      'void main(){'+
      '    gl_Position = vec4(p,0,1);'+
      '}');
  gl.compileShader(vshader);

  // Fragment shader: sample video texture, change colors
  const fshader = gl.createShader(gl.FRAGMENT_SHADER)!;
  gl.shaderSource(fshader,`
      precision highp float;

      uniform sampler2D data;
      uniform sampler2D data2;
      uniform float transitionProgress;
      uniform float rotationVelocity;
      uniform vec2 dataDimensions;
      uniform vec2 dataDimensions2;
      uniform int dataIsFacingUser;
      uniform float dataZoom;
      uniform vec2 canvasDimensions;
      uniform int scopeShape;
      uniform float scopeRotation;
      uniform float scopeSize;
      uniform vec2 scopeOffset;

      float round(float v) {
          return floor(v + 0.5);
      }

      vec2 rotate2d(vec2 u, float theta) {
        return vec2(
          cos(theta) * u.x - sin(theta) * u.y,
          sin(theta) * u.x + cos(theta) * u.y
        );
      }

      // https://observablehq.com/@jrus/hexround
      vec2 axial_round(vec2 pos) {
        float xGrid = round(pos.x);
        float yGrid = round(pos.y);
        pos.x -= xGrid; // remainder
        pos.y -= yGrid; // remainder
        float dx = round(pos.x + 0.5*pos.y) * (pos.x*pos.x >= pos.y*pos.y ? 1.0 : 0.0);
        float dy = round(pos.y + 0.5*pos.x) * (pos.x*pos.x < pos.y*pos.y ? 1.0 : 0.0);
        return vec2(xGrid + dx, yGrid + dy);
      }

      vec2 square_float_to_axial_hex_grid(vec2 pos, bool pointyTop) {
        float SQRT3 = sqrt(3.0);
        float ratio = 2.0 / 3.0;
        if (pointyTop) {
          pos.x = ratio * 0.5 * (SQRT3*pos.x - pos.y);
          pos.y *= ratio;
        } else {
          pos.y = ratio * 0.5 * (SQRT3*pos.y - pos.x);
          pos.x *= ratio;
        }
        return axial_round(pos);
      }

      vec2 hexToCentroid(vec2 hex, bool pointyTop) {
        vec2 pos = vec2(0.0,0.0);
        float size = 2.0 / 3.0;
        if (pointyTop) {
          pos.y = hex.y / size;
          pos.x = hex.x * 2.0 / sqrt(3.0) / size + hex.y / sqrt(3.0) / size;
        } else {
          pos.x = hex.x / size;
          pos.y = hex.y * 2.0 / sqrt(3.0) / size + hex.x / sqrt(3.0) / size;
        }
        return pos;
      }

      vec2 square(vec2 u, float kLength, float kRot, vec2 offset) {

        u -= vec2(0.5, 0.5);
        u = rotate2d(u,kRot);
        u /= kLength;
        u += vec2(0.5, 0.5);

        // Center the square in a circle
        u.x *= sqrt(2.0);
        u.y *= sqrt(2.0);
        u += vec2((1.0-sqrt(2.0))/2.0, (1.0-sqrt(2.0))/2.0);

        u += offset;

        float kOffset = (1.0/1.0 - 1.0) / (1.0/1.0 * 2.0);
        vec2 k = vec2(0.0, 0.0);
        if (mod(-kOffset + u.x, 1.0 * 2.0) > 1.0) {
          k.x = 1.0 - mod(-kOffset + u.x, 1.0) / 1.0;
        } else {
          k.x = mod(-kOffset + u.x, 1.0) / 1.0;
        }
        if (mod(-kOffset + u.y, 1.0 * 2.0) > 1.0) {
          k.y = 1.0 - mod(-kOffset + u.y, 1.0) / 1.0;
        } else {
          k.y = mod(-kOffset + u.y, 1.0) / 1.0;
        }
        return k;
      }

      vec2 equilateral(vec2 u, float kLength, float kRot, vec2 offset) {
        // Center the triangle in a circle

        u -= vec2(0.5, 0.5);

        u = rotate2d(u,kRot);
        u /= kLength;
        u /= sqrt(3.0) / 2.0;

        u += vec2(0.5, 0.5);
        u.y -= (0.25) * sqrt(3.0) / 2.0;

        u += offset;

        vec2 k = vec2(0.0, 0.0);

        vec2 hexIndex = square_float_to_axial_hex_grid(u, false);
        vec2 hexCentroid = hexToCentroid(hexIndex, false);
        // For debugging
        // k = vec2(distance(hexToCentroid(hexIndex, false), u));
        // k = hexIndex / 5.0;

        float distance = distance(hexCentroid, u);
        float deg180 = 3.1415926536;
        float deg60 = deg180 / 3.0;
        float theta = atan(u.x- hexCentroid.x, u.y- hexCentroid.y) + deg180 + deg60 / 2.0;
        if (mod(theta, deg60 * 2.0) > deg60) {
          theta = mod(theta, deg60);
        } else {
          theta = deg60 - mod(theta, deg60);
        }

        k.x = cos(theta) * distance;
        k.y = sin(theta) * distance;

        // We want the center of the equilateral triangle to be the center of the image.
        k *= cos(radians(30.0));
        k.x += (((1.0 - sqrt(3.0) / 2.0)) / 2.00);
        k.y += (0.254); // I don't know what number this actually is meant to be... But it works.

        return k;
      }


      void main() {
          // The interesting things to change!
          float kLength = scopeSize; // Effectively, how often should the image reflect 1 is one reflection (when viewed in square mode)
          float dataScopePercentage = 1.0; // Effectively how narrow the kaleidoscope should be, as a percentage of the camera size

          vec2 centerOffset = vec2(0.0);

          vec2 fragCoord = vec2(
            gl_FragCoord.x / canvasDimensions.x,
            gl_FragCoord.y / canvasDimensions.y
          );

          // Position in kaleidoscope space
          // In opengl, y-coordinate is flipped
          vec2 u = vec2(fragCoord.x,1.0-fragCoord.y);
          vec2 k = u;

          // Calculate tile distance from center BEFORE kaleidoscope transformation
          // This tells us which reflection copy we're in
          vec2 uCentered = u - vec2(0.5, 0.5);
          uCentered = rotate2d(uCentered, -scopeRotation);
          uCentered /= scopeSize; // Scale to tile space
          // Use circular/Euclidean distance to avoid star-shaped artifacts
          float tileDistance = length(uCentered);

          vec2 scopeOrigin = vec2(0.0, 0.0);
          float scopeDiameterRatio = sqrt(2.0);
          vec2 d = rotate2d(scopeOffset, 0.0);
          if (scopeShape == ${ScopeShape.Equilateral}) {
            k = equilateral(k, scopeSize, scopeRotation, d);
            scopeDiameterRatio = 1.0;
          } else if (scopeShape == ${ScopeShape.Square}) {
            k = square(k, scopeSize, scopeRotation, d);
            scopeDiameterRatio = sqrt(2.0);
          }

          // Calculate distance from edges of THIS reflection segment (using k coordinate)
          // k represents position within a single reflection segment, typically in [0,1] range
          // Calculate distance to nearest edge (more reliable than center distance)
          float distToEdgeX = min(k.x, 1.0 - k.x);
          float distToEdgeY = min(k.y, 1.0 - k.y);
          float distToNearestEdge = min(distToEdgeX, distToEdgeY);
          // Also calculate distance from center for radial effects
          vec2 segmentCenter = vec2(0.5, 0.5);
          vec2 toSegmentCenter = k - segmentCenter;
          float distFromSegmentCenter = length(toSegmentCenter);
          float maxSegmentDist = 0.707; // Maximum distance to corner
          
          // Progressive edge factor: smoothly transitions from 0 (center) to 1 (edges)
          // Use distance to nearest edge for smooth progressive effect
          float edgeFactorFromEdge = 1.0 - smoothstep(0.0, 0.5, distToNearestEdge); // Progressive from center to edge
          // Also use radial distance for smooth radial progression
          float edgeFactorFromCenter = smoothstep(0.0, maxSegmentDist, distFromSegmentCenter); // Progressive from center outward
          // Combine both for comprehensive edge detection - blend smoothly
          float edgeFactor = mix(edgeFactorFromCenter, edgeFactorFromEdge, 0.6);
          // Ensure smooth progressive transition (no sharp cutoffs)
          edgeFactor = smoothstep(0.0, 1.0, edgeFactor);
          
          // Now map the k value to coordinates on the image
          // 0,0 will be the centre of the image
          // 1,1 will be the top right of the image (not the bottom left– It's easier to orientate if things are up-right)
          float dataMinDimension = min(dataDimensions.x, dataDimensions.y) / scopeDiameterRatio;
          float dataWindowSize = dataMinDimension * dataScopePercentage;
          vec2 i = vec2(0.0,0.0);
          // x-axis is flipped only when the camera is pointed to the user
          i.x = (-dataWindowSize / 2.0 + k.x * dataWindowSize) * (dataIsFacingUser == 1 ? -1.0 : 1.0);
          // y-axis is flipped because of openGL coordinate space
          i.y = - (-dataWindowSize / 2.0 + k.y * dataWindowSize);
          i = rotate2d(i, scopeRotation * (dataIsFacingUser == 1 ? -1.0 : 1.0));
          i /= dataZoom;
          
          // Circular container: clipping is done by wrapper div (overflow hidden + rounded-full)
          vec2 screenCenter = vec2(0.5, 0.5);
          vec2 screenPosForMask = vec2(fragCoord.x, 1.0 - fragCoord.y); // Account for flipped y-coordinate
          float distFromCenter = distance(screenPosForMask, screenCenter);
          float circleRadius = 0.5; // Full extent (radius 0.5 = diameter 1) since div clips to circle
          
          // Radial vignette for circular container - progressively darken towards edges
          float normalizedDistFromCenter = distFromCenter / circleRadius; // Normalize to [0, 1] within circle
          // Fade vignette out before the mask edge to prevent bright halo
          // Stop vignette at 90% of radius so it fades before the mask transition
          float vignetteEnd = 0.9; // Stop vignette at 90% of circle radius
          float circleVignetteFactor = smoothstep(0.0, vignetteEnd, normalizedDistFromCenter); // Fade out before edge
          float circleVignette = 1.0 - circleVignetteFactor * 0.5; // Reduced darkening (50% darker at vignette end)
          circleVignette = max(circleVignette, 0.1); // Keep minimum brightness higher
          
          // Circular edge distortion - warp outward from circle center, stronger at edges
          // Use a curve that's minimal in center but aggressive at edges
          float circleDistortionFactor = smoothstep(0.0, 1.0, normalizedDistFromCenter); // Progressive from center to edge
          circleDistortionFactor = pow(circleDistortionFactor, 8.0); // Very steep curve - stays near zero until very close to edges, then ramps up dramatically
          
          // Distortion direction is radial from circle center (outward)
          vec2 circleDistortionDir = normalize(screenPosForMask - screenCenter + vec2(0.001)); // Avoid division by zero
          circleDistortionDir = rotate2d(circleDistortionDir, 0.1); // Slight rotation to smooth out patterns
          
          // Apply stronger distortion to texture coordinates based on circular container edge
          // Distortion pushes outward from center, creating a "lens" or "fisheye" effect at edges
          float circleDistortionStrength = circleDistortionFactor * 5.0; // Increased distortion strength
          vec2 circleDistortionOffset = circleDistortionDir * circleDistortionStrength * dataWindowSize * 0.03;
          
          // Chromatic aberration - only apply at circular container edges
          // Offset direction is radial from circle center (not segment center)
          // Only apply aberration when inside the circular container
          float aberrationStrength = circleDistortionFactor * 0.02; // Reduced aberration strength (in texture coordinate space)
          vec2 aberrationDir = normalize(rotate2d(screenPosForMask - screenCenter, scopeRotation) + vec2(0.001)); // Avoid division by zero
          
          vec2 iR = i + circleDistortionOffset + aberrationDir * aberrationStrength * dataWindowSize;
          vec2 iG = i + circleDistortionOffset;
          vec2 iB = i + circleDistortionOffset - aberrationDir * aberrationStrength * dataWindowSize;
          
          iR.x += dataDimensions.x / 2.0;
          iR.y += dataDimensions.y / 2.0;
          iG.x += dataDimensions.x / 2.0;
          iG.y += dataDimensions.y / 2.0;
          iB.x += dataDimensions.x / 2.0;
          iB.y += dataDimensions.y / 2.0;
          
          vec2 texCoordR = clamp(vec2(iR.x, iR.y) / vec2(dataDimensions.x, dataDimensions.y), 0.0, 1.0);
          vec2 texCoordG = clamp(vec2(iG.x, iG.y) / vec2(dataDimensions.x, dataDimensions.y), 0.0, 1.0);
          vec2 texCoordB = clamp(vec2(iB.x, iB.y) / vec2(dataDimensions.x, dataDimensions.y), 0.0, 1.0);
          
          // Motion blur disabled: remove spin blur on kaleidoscope rotation
          float motionBlurStrength = 0.0;
          
          vec3 color1R, color1G, color1B;
          float a1;
          
          // Only apply expensive motion blur when rotation velocity is high enough
          if (motionBlurStrength > 0.1) {
            // Calculate blur direction: tangential to rotation in texture space
            vec2 texCenter = vec2(0.5, 0.5);
            vec2 toTexCenter = texCoordG - texCenter;
            vec2 tangentDir = normalize(vec2(-toTexCenter.y, toTexCenter.x)) * sign(rotationVelocity);
            
            // Scale blur based on distance from center (stronger at edges)
            float distFromTexCenter = length(toTexCenter);
            float blurScale = smoothstep(0.0, 0.5, distFromTexCenter); // Stronger blur at edges
            motionBlurStrength *= blurScale;
            
            if (motionBlurStrength > 0.05) {
              // Apply motion blur - optimized with 3 samples
              vec3 blurR = vec3(0.0);
              vec3 blurG = vec3(0.0);
              vec3 blurB = vec3(0.0);
              float totalWeight = 0.0;
              
              for (int i = -1; i <= 1; i++) {
                float offset = float(i) * motionBlurStrength * 0.02;
                float weight = i == 0 ? 1.0 : 0.5;
                
                vec2 sampleR = clamp(texCoordR + tangentDir * offset, 0.0, 1.0);
                vec2 sampleG = clamp(texCoordG + tangentDir * offset, 0.0, 1.0);
                vec2 sampleB = clamp(texCoordB + tangentDir * offset, 0.0, 1.0);
                
                blurR += texture2D(data, sampleR).rgb * weight;
                blurG += texture2D(data, sampleG).rgb * weight;
                blurB += texture2D(data, sampleB).rgb * weight;
                totalWeight += weight;
              }
              
              if (totalWeight > 0.0) {
                color1R = blurR / totalWeight;
                color1G = blurG / totalWeight;
                color1B = blurB / totalWeight;
              } else {
                color1R = texture2D(data, texCoordR).rgb;
                color1G = texture2D(data, texCoordG).rgb;
                color1B = texture2D(data, texCoordB).rgb;
              }
              a1 = texture2D(data, texCoordG).a;
            } else {
              // No motion blur - direct sampling
              color1R = texture2D(data, texCoordR).rgb;
              color1G = texture2D(data, texCoordG).rgb;
              color1B = texture2D(data, texCoordB).rgb;
              a1 = texture2D(data, texCoordG).a;
            }
          } else {
            // No motion blur - direct sampling (fast path)
            color1R = texture2D(data, texCoordR).rgb;
            color1G = texture2D(data, texCoordG).rgb;
            color1B = texture2D(data, texCoordB).rgb;
            a1 = texture2D(data, texCoordG).a;
          }
          
          vec3 color1 = vec3(color1R.r, color1G.g, color1B.b);
          
          // Sample texture2 during transition - recalculate distortion for texture2's dimensions
          vec3 color2 = vec3(0.0);
          float a2 = 0.0;
          if (transitionProgress > 0.0) {
            // Recalculate image coordinates and distortion for texture2 using its dimensions
            float dataMinDimension2 = min(dataDimensions2.x, dataDimensions2.y) / scopeDiameterRatio;
            float dataWindowSize2 = dataMinDimension2 * dataScopePercentage;
            vec2 i2 = vec2(0.0,0.0);
            i2.x = (-dataWindowSize2 / 2.0 + k.x * dataWindowSize2) * (dataIsFacingUser == 1 ? -1.0 : 1.0);
            i2.y = - (-dataWindowSize2 / 2.0 + k.y * dataWindowSize2);
            i2 = rotate2d(i2, scopeRotation * (dataIsFacingUser == 1 ? -1.0 : 1.0));
            i2 /= dataZoom;
            
            // Apply same distortion calculations but with texture2's window size
            vec2 circleDistortionOffset2 = circleDistortionDir * circleDistortionStrength * dataWindowSize2 * 0.03;
            vec2 iR2 = i2 + circleDistortionOffset2 + aberrationDir * aberrationStrength * dataWindowSize2;
            vec2 iG2 = i2 + circleDistortionOffset2;
            vec2 iB2 = i2 + circleDistortionOffset2 - aberrationDir * aberrationStrength * dataWindowSize2;
            
            iR2.x += dataDimensions2.x / 2.0;
            iR2.y += dataDimensions2.y / 2.0;
            iG2.x += dataDimensions2.x / 2.0;
            iG2.y += dataDimensions2.y / 2.0;
            iB2.x += dataDimensions2.x / 2.0;
            iB2.y += dataDimensions2.y / 2.0;
            
            vec2 texCoordR2 = clamp(vec2(iR2.x, iR2.y) / vec2(dataDimensions2.x, dataDimensions2.y), 0.0, 1.0);
            vec2 texCoordG2 = clamp(vec2(iG2.x, iG2.y) / vec2(dataDimensions2.x, dataDimensions2.y), 0.0, 1.0);
            vec2 texCoordB2 = clamp(vec2(iB2.x, iB2.y) / vec2(dataDimensions2.x, dataDimensions2.y), 0.0, 1.0);
            
            // Simple direct sampling - image is already pre-rendered with all effects
            vec3 color2R = texture2D(data2, texCoordR2).rgb;
            vec3 color2G = texture2D(data2, texCoordG2).rgb;
            vec3 color2B = texture2D(data2, texCoordB2).rgb;
            color2 = vec3(color2R.r, color2G.g, color2B.b);
            a2 = texture2D(data2, texCoordG2).a;
          }
          
          // Blend between two images during transition
          vec3 finalColor1 = color1;
          vec3 finalColor2 = color2;
          
          // Edge blur - simplified for performance (only apply at very edges)
          float blurFactor = pow(circleDistortionFactor, 3.0); // More aggressive falloff
          
          // Only apply edge blur at very edges to save performance
          if (blurFactor > 0.8) {
            float blurStrength = (blurFactor - 0.8) * 0.04; // Only blur in last 20% of edge
            vec2 texCoordCenter = texCoordG;
            
            // Simplified blur - only 4 samples instead of 9 (unrolled for GLSL ES 1.0)
            vec3 blur1 = color1;
            
            vec2 sample1 = clamp(texCoordCenter + vec2(blurStrength, 0.0), 0.0, 1.0);
            vec2 sample2 = clamp(texCoordCenter + vec2(-blurStrength, 0.0), 0.0, 1.0);
            vec2 sample3 = clamp(texCoordCenter + vec2(0.0, blurStrength), 0.0, 1.0);
            vec2 sample4 = clamp(texCoordCenter + vec2(0.0, -blurStrength), 0.0, 1.0);
            
            blur1 += texture2D(data, sample1).rgb;
            blur1 += texture2D(data, sample2).rgb;
            blur1 += texture2D(data, sample3).rgb;
            blur1 += texture2D(data, sample4).rgb;
            
            blur1 /= 5.0; // Average with original
            
            float blendAmount = (blurFactor - 0.8) * 5.0; // Scale to 0-1
            finalColor1 = mix(color1, blur1, blendAmount);
          }
          
          // Blend between images with smooth transition - simple opacity blend
          float blendFactor = smoothstep(0.0, 1.0, transitionProgress); // Smooth easing
          vec3 finalColor = mix(finalColor1, color2, blendFactor);
          float finalAlpha = mix(a1, a2, blendFactor);
          
          // Limit reflections to main and adjacent tiles only (no infinite reflections)
          // tileDistance < 0.5 = main reflection, 0.5 <= tileDistance < 1.5 = adjacent reflections
          float maxTileDistance = 1.5; // Maximum tile distance to show
          float tileMask = 1.0 - smoothstep(maxTileDistance - 0.1, maxTileDistance, tileDistance); // Smooth edge
          
          // Apply vignette and tile mask (circle clipping is done by wrapper div)
          float combinedMask = tileMask;
          gl_FragColor = vec4(finalColor * circleVignette, finalAlpha * combinedMask);

          // For debugging the kaleidoscope value
          // gl_FragColor=vec4(k.x, k.y, 0.0, 1.0);
      }`);
  gl.compileShader(fshader);

  // Create and link program
  const program  = gl.createProgram()!;
  gl.attachShader(program,vshader);
  gl.attachShader(program,fshader);
  gl.linkProgram(program);
  gl.useProgram(program);

  // Vertices: A screen-filling quad made from two triangles
  gl.bindBuffer(gl.ARRAY_BUFFER, gl.createBuffer());
  gl.bufferData(gl.ARRAY_BUFFER,new Float32Array([-1,-1,1,-1,-1,1,-1,1,1,-1,1,1]),gl.STATIC_DRAW);
  gl.enableVertexAttribArray(0);
  gl.vertexAttribPointer(0, 2, gl.FLOAT, false, 0, 0);

  // Textures to contain the video/image data
  texture1 = gl.createTexture();
  texture2 = gl.createTexture();
  
  const setupTexture = (tex: WebGLTexture | null) => {
    if (!tex) return;
    gl.bindTexture(gl.TEXTURE_2D, tex);
    gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, true);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
    gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
  };
  
  setupTexture(texture1);
  setupTexture(texture2);

  // Bind textures to the fragment shader
  gl.uniform1i(gl.getUniformLocation(program,'data'),0);
  gl.uniform1i(gl.getUniformLocation(program,'data2'),1);
  gl.activeTexture(gl.TEXTURE0);
  gl.bindTexture(gl.TEXTURE_2D, texture1);
  gl.activeTexture(gl.TEXTURE1);
  gl.bindTexture(gl.TEXTURE_2D, texture2);

  // Bind camera dimensions to the fragment shader
  const dataDimensionsBind = gl.getUniformLocation(program, 'dataDimensions');
  const dataDimensions2Bind = gl.getUniformLocation(program, 'dataDimensions2');
  const dataIsFacingUserBind = gl.getUniformLocation(program, 'dataIsFacingUser');
  const dataZoomBind = gl.getUniformLocation(program, 'dataZoom');
  const canvasDimensionsBind = gl.getUniformLocation(program, 'canvasDimensions');
  const scopeShapeBind = gl.getUniformLocation(program, 'scopeShape');
  const scopeRotationBind = gl.getUniformLocation(program, 'scopeRotation');
  const scopeSizeBind = gl.getUniformLocation(program, 'scopeSize');
  const scopeOffsetBind = gl.getUniformLocation(program, 'scopeOffset');
  const rotationVelocityBind = gl.getUniformLocation(program, 'rotationVelocity');
  const transitionProgressBind = gl.getUniformLocation(program, 'transitionProgress');

  function fastNormalSlow(fast: number, normal: number, slow: number) {
    if (keyPressedShift.value) {
      return fast;
    }
    if (keyPressedAlt.value) {
      return slow;
    }
    return normal;
  }

  // Repeatedly pull camera data and render
  const renderToDisplayCanvas = (displayCanvas: HTMLCanvasElement | null, sourceCanvas: HTMLCanvasElement) => {
    if (!displayCanvas) {
      return;
    }
    const ctx = displayCanvas.getContext('2d');
    if (!ctx) {
      return;
    }
    const dpr = window.devicePixelRatio || 1;
    const targetWidth = Math.max(1, Math.floor(displayCanvas.clientWidth * dpr));
    const targetHeight = Math.max(1, Math.floor(displayCanvas.clientHeight * dpr));
    if (displayCanvas.width !== targetWidth || displayCanvas.height !== targetHeight) {
      displayCanvas.width = targetWidth;
      displayCanvas.height = targetHeight;
    }
    ctx.setTransform(1, 0, 0, 1, 0, 0);
    ctx.clearRect(0, 0, targetWidth, targetHeight);
    ctx.drawImage(sourceCanvas, 0, 0, targetWidth, targetHeight);
  };

  const drawOrbFrame = (
    displayCanvas: HTMLCanvasElement | null,
    imageSource: HTMLImageElement | HTMLVideoElement,
    scopeScaleMultiplier: number
  ) => {
    const isVideo = imageSource instanceof HTMLVideoElement;
    const imageWidth = isVideo ? imageSource.videoWidth : imageSource.width;
    const imageHeight = isVideo ? imageSource.videoHeight : imageSource.height;
    if (imageWidth <= 0 || imageHeight <= 0) {
      return;
    }

    gl.activeTexture(gl.TEXTURE0);
    gl.bindTexture(gl.TEXTURE_2D, texture1);
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, imageSource);

    gl.uniform2f(dataDimensionsBind, imageWidth, imageHeight);
    gl.uniform2f(dataDimensions2Bind, imageWidth, imageHeight);
    gl.uniform1f(dataZoomBind, cameraZoom.value);
    gl.uniform1f(scopeSizeBind, scopeSize.value * scopeScaleMultiplier);
    if (transitionProgressBind) {
      gl.uniform1f(transitionProgressBind, 0.0);
    }
    gl.drawArrays(gl.TRIANGLES, 0, 6);
    renderToDisplayCanvas(displayCanvas, canvasElement);
  };

  let lastAnimateTime = performance.now();
// Throttle rapid index changes so interrupting a snap doesn't cycle images wildly
let lastOrbIndexChange = 0;
const ORB_INDEX_CHANGE_MIN_MS = 80;
  function animate(){
    const now = performance.now();
    const dt = Math.min(0.032, (now - lastAnimateTime) / 1000);
    lastAnimateTime = now;

    // Integrate orb scroll physics (skip while fill animation is running)
    if (!orbFillAnimating.value && Math.abs(orbScrollVel.value) > 0) {
      orbScrollOffset.value += orbScrollVel.value * dt;
      // Apply exponential friction for frame-rate independence
      orbScrollVel.value *= Math.exp(-ORB_FRICTION * dt);
      if (Math.abs(orbScrollVel.value) < ORB_VELOCITY_THRESHOLD) {
        orbScrollVel.value = 0;
      }

      // Handle crossing integer boundaries (advance active index)
      const totalImagesBoundary = uploadedImageElements.value.length;
    // Only consume at most one index step per short time window to avoid rapid cycling
    if (orbScrollOffset.value >= 1 || orbScrollOffset.value <= -1) {
      const sign = orbScrollOffset.value >= 1 ? 1 : -1;
      const nowIdx = performance.now();
      if (nowIdx - lastOrbIndexChange >= ORB_INDEX_CHANGE_MIN_MS) {
        if (totalImagesBoundary > 0) {
          activeImageIndex.value = getWrappedIndex(activeImageIndex.value + sign, totalImagesBoundary);
        }
        orbScrollOffset.value -= sign;
        lastOrbIndexChange = nowIdx;
      } else {
        // If we're inside the cooldown, clamp offset just below the integer boundary
        // so we don't repeatedly trigger index changes until cooldown elapses.
        const clamped = Math.sign(orbScrollOffset.value) * Math.min(0.999, Math.abs(orbScrollOffset.value));
        orbScrollOffset.value = clamped;
      }
    }

      syncSlotIndices();
      orbTransitionDirection.value = orbScrollOffset.value >= 0 ? 1 : -1;
      orbTransitionProgress.value = Math.min(1, Math.abs(orbScrollOffset.value));
      isOrbTransitioning.value = orbTransitionProgress.value > 0.001;
    } else if (!orbFillAnimating.value) {
      // If velocity is zero and user isn't interacting, optionally start snap to nearest
      if (!isUserPressing.value && Math.abs(orbScrollOffset.value - Math.round(orbScrollOffset.value)) > 0.001 && orbSnapRaf === null) {
        // startOrbSnap will handle small spring settling
        startOrbSnap();
      }
    }

    // Handle keyboard state
    if (keyPressedA.value && keyPressedD.value) {
      // Do nothing
    } else if (keyPressedA.value) {
      scopeRotationVel.value = clampRotationVelocity(scopeRotationVel.value - fastNormalSlow(0.001, 0.0002, 0.00005));
    } else if (keyPressedD.value) {
      scopeRotationVel.value = clampRotationVelocity(scopeRotationVel.value + fastNormalSlow(0.001, 0.0002, 0.00005));
    }
    if (keyPressedW.value && keyPressedS.value) {
      // Do nothing
    } else if (keyPressedW.value) {
      scopeSizeVel.value += fastNormalSlow(0.001, 0.0002, 0.00005);
    } else if (keyPressedS.value) {
      scopeSizeVel.value -= fastNormalSlow(0.001, 0.0002, 0.00005);
    }
    if (keyPressedI.value && keyPressedK.value) {
      // Do nothing
    } else if (keyPressedI.value) {
      scopeOffsetVel.value[0] -= 0.0002 / scopeSize.value;
    } else if (keyPressedK.value) {
      scopeOffsetVel.value[0] += 0.0002 / scopeSize.value;
    }
    if (keyPressedJ.value && keyPressedL.value) {
      // Do nothing
    } else if (keyPressedJ.value) {
      scopeOffsetVel.value[1] += 0.0002 / scopeSize.value;
    } else if (keyPressedL.value) {
      scopeOffsetVel.value[1] -= 0.0002 / scopeSize.value;
    }
    if (keyPressedMinus.value && keyPressedPlus.value) {
      // Do nothing
    } else if (keyPressedMinus.value) {
      cameraZoom.value = Math.max(1, cameraZoom.value * 0.95);
    } else if (keyPressedPlus.value) {
      cameraZoom.value = Math.min(10, cameraZoom.value * 1.05);
    }

    let scopeRotationOffset = 0;
    if (props.scopeShape === ScopeShape.Equilateral) {
      scopeRotationOffset = Math.PI / 3;
    }

    scopeOffset.value[0] += Math.sin(-scopeRotation.value - scopeRotationOffset) * scopeOffsetVel.value[0] - Math.cos(-scopeRotation.value - scopeRotationOffset) * scopeOffsetVel.value[1];
    scopeOffset.value[1] += Math.cos(-scopeRotation.value - scopeRotationOffset) * scopeOffsetVel.value[0] + Math.sin(-scopeRotation.value - scopeRotationOffset) * scopeOffsetVel.value[1];

    if (props.scopeAutoRotationVelocity !== 0) {
      // Apply base auto rotation to center orb; top/bottom receive independent multipliers
      const baseAuto = clampRotationVelocity(props.scopeAutoRotationVelocity / 25);
      scopeRotationVel.value = baseAuto;
      scopeRotationTopVel.value = clampRotationVelocity(baseAuto * 0.8);
      scopeRotationBottomVel.value = clampRotationVelocity(baseAuto * 1.2);
      scopeRotation.value += scopeRotationVel.value;
    }
    if (touchOrigin1 === null && mousePrevPosition === null) {
      // Update center orb rotation and damp velocity
      scopeRotation.value += scopeRotationVel.value;
      scopeRotationVel.value = clampRotationVelocity(scopeRotationVel.value * 0.99);
      // Update top and bottom orb rotations independently with light damping
      scopeRotationTop.value += scopeRotationTopVel.value;
      scopeRotationTopVel.value = clampRotationVelocity(scopeRotationTopVel.value * 0.995);
      scopeRotationBottom.value += scopeRotationBottomVel.value;
      scopeRotationBottomVel.value = clampRotationVelocity(scopeRotationBottomVel.value * 0.995);
      scopeSizeVel.value *= 0.95;
      scopeSize.value = Math.max(0.5, Math.min(1, scopeSize.value * (1 + Math.min(scopeSizeVel.value, 0.99))));
    }
    scopeOffsetVel.value[0] *= 0.95;
    scopeOffsetVel.value[1] *= 0.95;

    if (orbTransitionProgress.value <= 0.001) {
      isOrbTransitioning.value = false;
      orbTransitionProgress.value = 0.0;
    }

    const hasUploadedImages = uploadedImageElements.value.length > 0;
    const topImage = hasUploadedImages ? uploadedImageElements.value[topSlotIndex.value] : null;
    const centerImage = hasUploadedImages ? uploadedImageElements.value[centerSlotIndex.value] : null;
    const bottomImage = hasUploadedImages ? uploadedImageElements.value[bottomSlotIndex.value] : null;
    const totalImages = uploadedImageElements.value.length;
    const incomingIndex = totalImages > 0
      ? getWrappedIndex(activeImageIndex.value + (orbScrollOffset.value >= 0 ? 2 : -2), totalImages)
      : 0;
    const incomingImage = totalImages > 0 ? uploadedImageElements.value[incomingIndex] : null;
    const cameraFallback = camera;
    const centerSource = getReadySource(centerImage, cameraFallback);
    const topSource = getReadySource(topImage, centerSource);
    const bottomSource = getReadySource(bottomImage, centerSource);
    const incomingSource = getReadySource(incomingImage, centerSource);

    // Shared uniforms
    gl.uniform1i(dataIsFacingUserBind, facingMode.value === 'user' ? 1 : 0);
    gl.uniform1i(scopeShapeBind, props.scopeShape);
    gl.uniform2f(scopeOffsetBind, scopeOffset.value[0], scopeOffset.value[1]);
    gl.uniform2f(canvasDimensionsBind, canvasSize, canvasSize);

    // Top orb: use independent rotation & velocity
    gl.uniform1f(scopeRotationBind, scopeRotationTop.value + scopeRotationOffset);
    gl.uniform1f(rotationVelocityBind, scopeRotationTopVel.value);
    drawOrbFrame(displayCanvasTop.value ?? null, topSource ?? cameraFallback, getOrbScopeScale('top'));

    // Incoming orb (during transitions) — use top orb's rotation for continuity
    gl.uniform1f(scopeRotationBind, scopeRotationTop.value + scopeRotationOffset);
    gl.uniform1f(rotationVelocityBind, scopeRotationTopVel.value);
    drawOrbFrame(displayCanvasIncoming.value ?? null, incomingSource ?? cameraFallback, getOrbScopeScale('incoming'));

    // Bottom orb: independent rotation & velocity
    gl.uniform1f(scopeRotationBind, scopeRotationBottom.value + scopeRotationOffset);
    gl.uniform1f(rotationVelocityBind, scopeRotationBottomVel.value);
    drawOrbFrame(displayCanvasBottom.value ?? null, bottomSource ?? cameraFallback, getOrbScopeScale('bottom'));

    // Center orb: user-controlled rotation
    gl.uniform1f(scopeRotationBind, scopeRotation.value + scopeRotationOffset);
    gl.uniform1f(rotationVelocityBind, scopeRotationVel.value);
    drawOrbFrame(displayCanvasCenter.value ?? null, centerSource ?? cameraFallback, getOrbScopeScale('center'));

    if (props.saveNextFrame) {
      emit(
        'save-frame',
        canvasElement.toDataURL('image/jpeg', 0.8)
      );
    }

    requestAnimationFrame(animate);
  }
  animate();
}

interface Point {
  clientX: number;
  clientY: number;
}
let mousePrevPosition = null as null|{x: number, y: number};
let mouseStartPosition = null as null|{x: number, y: number};
let touchId1: null|number = null;
let touchOrigin1: null|Point = null;
let touchPrevTime = new Date().getTime();
let touchPrev1: null|Point = null;
let pinchPrevDist: null|number = null;

function getTouchById(touches: TouchList, id: number): Touch | null {
  for (let i = 0; i < touches.length; i += 1) {
    if (touches[i].identifier === id) {
      return touches[i];
    }
  }
  return null;
}

function touchDistance(t1: Touch, t2: Touch): number {
  return Math.hypot(t2.clientX - t1.clientX, t2.clientY - t1.clientY);
}

// Helper: determine if a point/event is inside the center orb element
type PointLike = { clientX?: number; clientY?: number; x?: number; y?: number };
const isPointInsideCenterOrb = (p: PointLike | Touch | MouseEvent): boolean => {
  const obj = p as PointLike;
  let clientX = 0;
  let clientY = 0;
  if (typeof obj.clientX === 'number') {
    clientX = obj.clientX;
  } else if (typeof obj.x === 'number') {
    clientX = obj.x;
  }
  if (typeof obj.clientY === 'number') {
    clientY = obj.clientY;
  } else if (typeof obj.y === 'number') {
    clientY = obj.y;
  }
  const el = centerOrb.value as HTMLDivElement | undefined;
  if (!el) return false;
  const rect = el.getBoundingClientRect();
  const dx = clientX - (rect.left + rect.width / 2);
  const dy = clientY - (rect.top + rect.height / 2);
  return Math.hypot(dx, dy) <= Math.min(rect.width, rect.height) / 2;
};

// Reset drag visuals for center orb
const resetOrbDragVisuals = () => {
  orbDragTranslateX.value = 0;
  orbDragTranslateY.value = 0;
  orbDragScaleMultiplier.value = 1;
  isOrbDragging.value = false;
  isOrbDismissing.value = false;
  pendingOrbRemovalIndex.value = null;
  orbDragLastX = null;
};

// Set drag visuals while dragging
const setOrbDragVisuals = (deltaX: number) => {
  orbDragTranslateX.value = deltaX;
  orbDragTranslateY.value = 0;
  // modest scale down while dragging
  orbDragScaleMultiplier.value = Math.max(0.85, 1 - Math.abs(deltaX) / 1000);
};

const getCenterOrbDismissThresholdPx = (): number => {
  return Math.max(60, Math.round(Math.min(window.innerWidth, window.innerHeight) * 0.15));
};

const dismissActiveOrb = async (direction: 1 | -1, removeIndex: number) => {
  if (uploadedImageElements.value.length === 0) return;
  isOrbDismissing.value = true;
  pendingOrbRemovalIndex.value = removeIndex;
  // animate by setting translate and scale
  orbDragTranslateX.value = direction * (getCenterOrbDismissThresholdPx() + 20);
  orbDragScaleMultiplier.value = 0.85;
  // First, wait for the horizontal dismiss visual to complete (center orb moves offscreen).
  await new Promise<void>(resolve => setTimeout(() => resolve(), ORB_DISMISS_DURATION_MS));

  // Hide the center DOM so it does not immediately reappear with a different photo.
  isCenterHiddenDuringDismiss.value = true;

  // Prevent any ongoing scroll physics/snapping from interfering with the fill animation.
  orbScrollVel.value = 0;
  cancelOrbSnap();
  // Animate the vertical orb-stack fill (other orbs move into center). This is strictly vertical.
  await animateOrbStackFill(direction, ORB_DISMISS_DURATION_MS);

  // Perform local removal so UI updates after the fill animation completes.
  removeUploadedImageAtIndex(removeIndex);
  // Notify parent about removal (watch handler will detect pending removal and avoid double-removal).
  emit('remove-uploaded-image', removeIndex);

  // Reveal center orb now that the vertical fill animation and local removal completed.
  isCenterHiddenDuringDismiss.value = false;

  // cleanup
  isOrbDismissing.value = false;
  resetOrbDragVisuals();
};

// Animate the orb stack so top/center/bottom visually shift to fill a removed center orb.
// direction: 1 => bottom moves into center (scroll up), -1 => top moves into center (scroll down)
const animateOrbStackFill = (direction: 1 | -1, durationMs: number) => {
  return new Promise<void>((resolve) => {
    const start = performance.now();
    const from = 0;
    const to = direction;
    // easing
    const easeOutCubic = (t: number) => 1 - Math.pow(1 - t, 3);

    isOrbTransitioning.value = true;
    orbTransitionDirection.value = direction;
    orbFillAnimating.value = true;

    const step = (now: number) => {
      const elapsed = Math.min(durationMs, now - start);
      const t = durationMs > 0 ? elapsed / durationMs : 1;
      const eased = easeOutCubic(t);
      orbScrollOffset.value = lerp(from, to, eased);
      orbTransitionDirection.value = orbScrollOffset.value >= 0 ? 1 : -1;
      orbTransitionProgress.value = Math.min(1, Math.abs(orbScrollOffset.value));

      if (elapsed >= durationMs) {
        // finish animation
        orbScrollOffset.value = 0; // reset offset; actual new center will be set by local removal & syncSlotIndices
        orbTransitionProgress.value = 0;
        isOrbTransitioning.value = false;
        orbFillAnimating.value = false;
        resolve();
        return;
      }
      requestAnimationFrame(step);
    };
    requestAnimationFrame(step);
  });
};

const removeUploadedImageAtIndex = (index: number) => {
  if (index < 0 || index >= uploadedImageElements.value.length) return;
  uploadedImageElements.value.splice(index, 1);
  if (activeImageIndex.value >= uploadedImageElements.value.length) {
    activeImageIndex.value = Math.max(0, uploadedImageElements.value.length - 1);
  }
  syncSlotIndices();
};

function touchStartCallback(event: TouchEvent) {
  event.preventDefault();
  const len = event.touches.length;
  if (len === 2) {
    orbDragTouchActive = false;
    orbDragTouchLast = null;
    isOrbDragging.value = false;
    resetOrbDragVisuals();
    touchId1 = null;
    touchPrev1 = null;
    touchOrigin1 = null;
    pinchPrevDist = touchDistance(event.touches[0], event.touches[1]);
    scopeSizeVel.value = 0;
    return;
  }
  if (len === 1 && touchId1 === null && pinchPrevDist === null) {
    const touch = event.changedTouches[0];
    // If user starts within the center orb and we have an uploaded stack, treat horizontal drag as dismiss gesture.
    if (
      uploadedImageElements.value.length > 0 &&
      !isOrbTransitioning.value &&
      !isOrbDismissing.value &&
      isPointInsideCenterOrb(touch)
    ) {
      orbDragTouchActive = true;
      orbDragTouchLast = touch;
      orbDragLastX = touch.clientX;
      isOrbDragging.value = true;
    } else {
      orbDragTouchActive = false;
      orbDragTouchLast = null;
      isOrbDragging.value = false;
    }
    touchId1 = touch.identifier;
    touchPrev1 = touch;
    touchOrigin1 = touch;
    touchPrevTime = new Date().getTime();
    isUserPressing.value = true;
    scopeRotationVel.value = 0;
    scopeSizeVel.value = 0;
  }
}

function touchMoveCallback(event: TouchEvent) {
  event.preventDefault();
  const len = event.touches.length;
  if (len === 2 && pinchPrevDist !== null) {
    const dist = touchDistance(event.touches[0], event.touches[1]);
    const ratio = dist / pinchPrevDist;
    scopeSize.value = Math.max(0.5, Math.min(1, scopeSize.value * ratio));
    scopeSizeVel.value = clampScopeSizeVelocity(scopeSizeVel.value + (ratio - 1) * 0.6);
    pinchPrevDist = dist;
    return;
  }
  if (len === 1 && touchId1 !== null && touchPrev1 !== null && touchOrigin1 !== null) {
    const touch = getTouchById(event.touches, touchId1);
    if (touch === null) {
      return;
    }
    if (orbDragTouchActive) {
      // Rotate kaleidoscope incrementally based on horizontal drag delta
      const now = performance.now();
      const prevX = orbDragLastX ?? touch.clientX;
      const dx = touch.clientX - prevX;
      const dt = Math.max(1, now - touchPrevTime);
      // Sensitivity tuned for pleasant feel
      scopeRotation.value += dx / 300;
      scopeRotationVel.value = clampRotationVelocity((dx / dt) * 0.02);
      orbDragLastX = touch.clientX;
      orbDragTouchLast = touch;
      setOrbDragVisuals(touch.clientX - touchOrigin1.clientX);
      touchPrevTime = now;
      return;
    }
    if (new Date().getTime() - touchPrevTime < 0.001) {
      return;
    }
    const deltaTime = Math.max(new Date().getTime() - touchPrevTime, 0.001);
    const deltaX = (touch.clientX - touchPrev1.clientX) / deltaTime;

    scopeRotation.value -= deltaX / 22;
    if (Math.abs(touch.clientX - touchPrev1.clientX) > 1) {
      scopeRotationVel.value = clampRotationVelocity(-deltaX / 22);
    } else {
      scopeRotationVel.value = 0;
    }

    touchPrevTime = new Date().getTime();
    touchPrev1 = touch;
  }
}

function touchEndCallback(event: TouchEvent) {
  const len = event.touches.length;
  if (len === 2) {
    pinchPrevDist = touchDistance(event.touches[0], event.touches[1]);
    return;
  }
  if (len === 1) {
    pinchPrevDist = null;
    const remaining = event.touches[0];
    touchId1 = remaining.identifier;
    touchPrev1 = remaining;
    touchOrigin1 = remaining;
    touchPrevTime = new Date().getTime();
    isUserPressing.value = true;
  }
  if (len === 0) {
    if (orbDragTouchActive && touchOrigin1 !== null) {
      const last = orbDragTouchLast ?? (touchId1 !== null ? getTouchById(event.changedTouches, touchId1) : null);
      const deltaX = last ? last.clientX - touchOrigin1.clientX : orbDragTranslateX.value;
      const threshold = getCenterOrbDismissThresholdPx();
      const direction: 1 | -1 = deltaX >= 0 ? 1 : -1;
      const removeIndex = centerSlotIndex.value;

      orbDragTouchActive = false;
      orbDragTouchLast = null;
      isOrbDragging.value = false;

      if (Math.abs(deltaX) >= threshold && uploadedImageElements.value.length > 0) {
        void dismissActiveOrb(direction, removeIndex);
      } else {
        resetOrbDragVisuals();
      }
    } else if (touchId1 !== null && touchOrigin1 !== null) {
      const touch = getTouchById(event.changedTouches, touchId1);
      if (touch !== null) {
        const deltaX = touch.clientX - touchOrigin1.clientX;
        const deltaY = touch.clientY - touchOrigin1.clientY;
        const dist = Math.hypot(deltaX, deltaY);
        const didSwipe = handleSwipeGesture(deltaX, deltaY);
        if (!didSwipe && dist < CLICK_MOVE_THRESHOLD_PX) {
          emit('upload-click');
        }
      }
    }
    // Snap to nearest orb after touch interaction ends
    startOrbSnap();
    isUserPressing.value = false;
    touchId1 = null;
    touchPrev1 = null;
    touchOrigin1 = null;
    pinchPrevDist = null;
    return;
  }
  if (touchId1 === null || touchPrev1 === null || touchOrigin1 === null) {
    return;
  }
  const touch = getTouchById(event.changedTouches, touchId1);
  if (touch === null) {
    return;
  }
  // Snap after touch interaction end (non-zero-changed-touches branch)
  startOrbSnap();
  isUserPressing.value = false;
  touchId1 = null;
  touchPrev1 = null;
  touchOrigin1 = null;
}

function touchCancelCallback() {
  isUserPressing.value = false;
  touchId1 = null;
  touchPrev1 = null;
  touchOrigin1 = null;
  pinchPrevDist = null;
}

onMounted(async () => {
  await nextTick();
  const canvasElement = canvas.value as HTMLCanvasElement | undefined;
  if (!canvasElement) {
    console.error('Kaleidoscope: canvas ref not available');
    return;
  }
  main(canvasElement);
  const interactionElement = interactionLayer.value as HTMLDivElement | undefined;
  if (!interactionElement) {
    console.error('Kaleidoscope: interaction layer ref not available');
    return;
  }

  interactionElement.addEventListener('mousedown', (mouseEvent) => {
    // Left mouse button only
    if (mouseEvent.button !== 0) {
      return;
    }
    // Prevent default dragging/selection behavior so drag only controls rotation
    mouseEvent.preventDefault();
    const pos = { x: mouseEvent.clientX, y: mouseEvent.clientY };
    const shouldOrbDrag =
      uploadedImageElements.value.length > 0 &&
      !isOrbTransitioning.value &&
      !isOrbDismissing.value &&
      isPointInsideCenterOrb(mouseEvent);
    if (shouldOrbDrag) {
      orbDragMouseActive = true;
      orbDragMouseStart = pos;
      orbDragLastX = pos.x;
      isOrbDragging.value = true;
      mousePrevPosition = null;
      mouseStartPosition = null;
    } else {
      orbDragMouseActive = false;
      orbDragMouseStart = null;
      isOrbDragging.value = false;
      mousePrevPosition = pos;
      mouseStartPosition = pos;
    }
    isUserPressing.value = true;
    scopeRotationVel.value = 0;
    scopeSizeVel.value = 0;
  });
  let mousePrevTime = performance.now();
  document.addEventListener('mousemove', (mouseEvent) => {
    if (orbDragMouseActive && orbDragMouseStart !== null) {
      // Rotate kaleidoscope incrementally based on horizontal drag delta
      const now = performance.now();
      const prevX = orbDragLastX ?? mouseEvent.clientX;
      const dx = mouseEvent.clientX - prevX;
      const dt = Math.max(1, now - mousePrevTime);
      scopeRotation.value += dx / 300;
      scopeRotationVel.value = clampRotationVelocity((dx / dt) * 0.02);
      orbDragLastX = mouseEvent.clientX;
      setOrbDragVisuals(mouseEvent.clientX - orbDragMouseStart.x);
      mousePrevTime = now;
      return;
    }
    if (mousePrevPosition === null) {
      return;
    }
    const now = performance.now();
    const deltaTime = Math.max(1, now - mousePrevTime);
    const deltaX = (mouseEvent.clientX - mousePrevPosition.x) / 10;

    scopeRotation.value += deltaX / 35;
    if (Math.abs(mouseEvent.clientX - mousePrevPosition.x) > 1) {
      scopeRotationVel.value = clampRotationVelocity((deltaX / deltaTime) * 0.35);
    } else {
      scopeRotationVel.value = 0;
    }

    mousePrevPosition = {
      x: mouseEvent.clientX,
      y: mouseEvent.clientY,
    };
    mousePrevTime = now;
  });
  document.addEventListener('mouseup', (mouseEvent: MouseEvent) => {
    if (orbDragMouseActive && orbDragMouseStart !== null) {
      const deltaX = mouseEvent.clientX - orbDragMouseStart.x;
      const threshold = getCenterOrbDismissThresholdPx();
      const direction: 1 | -1 = deltaX >= 0 ? 1 : -1;
      const removeIndex = centerSlotIndex.value;

      orbDragMouseActive = false;
      orbDragMouseStart = null;
      isOrbDragging.value = false;

      if (Math.abs(deltaX) >= threshold && uploadedImageElements.value.length > 0) {
        void dismissActiveOrb(direction, removeIndex);
      } else {
        resetOrbDragVisuals();
      }

      isUserPressing.value = false;
      return;
    }
    if (mousePrevPosition === null || mouseStartPosition === null) {
      return;
    }
    const deltaX = mouseEvent.clientX - mouseStartPosition.x;
    const deltaY = mouseEvent.clientY - mouseStartPosition.y;
    const dist = Math.hypot(deltaX, deltaY);
    const didSwipe = handleSwipeGesture(deltaX, deltaY);
    if (!didSwipe && dist < CLICK_MOVE_THRESHOLD_PX) {
      emit('upload-click');
    }
    // Snap to nearest orb after mouse interaction ends
    startOrbSnap();
    isUserPressing.value = false;
    mousePrevPosition = null;
    mouseStartPosition = null;
  });

  document.addEventListener('wheel', (wheelEvent) => {
    wheelEvent.preventDefault();
    if (Math.abs(wheelEvent.deltaY) < 4) {
      return;
    }
    const clampedDelta = Math.max(-120, Math.min(120, wheelEvent.deltaY));
    const isTrackpad = wheelEvent.deltaMode === 0;
    // Compute impulse from raw delta (single mapping, nonlinear)
    const progressDelta = computeImpulseFromDelta(clampedDelta, isTrackpad);
    // Track last wheel delta to detect fast flicks (use raw clamped value)
    try {
      orbLastWheelDelta = clampedDelta;
    } catch {
      // silent
    }
    applyOrbScrollDelta(progressDelta);
    // Snap after a short pause in wheel activity
    if (orbSnapTimeout !== null) {
      clearTimeout(orbSnapTimeout);
    }
    orbSnapTimeout = window.setTimeout(() => {
      orbSnapTimeout = null;
      startOrbSnap();
    }, 150);
  }, { passive: false });

  // Use non-passive touch listeners so preventDefault() in handlers stops page scrolling
  interactionElement.addEventListener('touchstart', touchStartCallback, { passive: false });
  interactionElement.addEventListener('touchmove', touchMoveCallback, { passive: false });
  interactionElement.addEventListener('touchend', touchEndCallback, { passive: false });
  interactionElement.addEventListener('touchcancel', touchCancelCallback, { passive: false });

  document.addEventListener('keydown', (keyEvent) => {
    if (keyEvent.code == 'KeyW') {
      keyPressedW.value = true;
    } else if (keyEvent.code == 'KeyA') {
      keyPressedA.value = true;
    } else if (keyEvent.code == 'KeyS') {
      keyPressedS.value = true;
    } else if (keyEvent.code == 'KeyD') {
      keyPressedD.value = true;
    } else if (keyEvent.code == 'KeyI') {
      keyPressedI.value = true;
    } else if (keyEvent.code == 'KeyJ') {
      keyPressedJ.value = true;
    } else if (keyEvent.code == 'KeyK') {
      keyPressedK.value = true;
    } else if (keyEvent.code == 'KeyL') {
      keyPressedL.value = true;
    } else if (keyEvent.code == 'Equal' && !keyEvent.metaKey && !keyEvent.ctrlKey) {
      keyPressedPlus.value = true;
    } else if (keyEvent.code == 'Minus' && !keyEvent.metaKey && !keyEvent.ctrlKey) {
      keyPressedMinus.value = true;
    } else if (keyEvent.code == 'ShiftLeft' || keyEvent.code == 'ShiftRight') {
      keyPressedShift.value = true;
    } else if (keyEvent.code == 'AltLeft' || keyEvent.code == 'AltRight') {
      keyPressedAlt.value = true;
    }
  });
  document.addEventListener('keyup', (keyEvent) => {
    if (keyEvent.code == 'KeyW') {
      keyPressedW.value = false;
    } else if (keyEvent.code == 'KeyA') {
      keyPressedA.value = false;
    } else if (keyEvent.code == 'KeyS') {
      keyPressedS.value = false;
    } else if (keyEvent.code == 'KeyD') {
      keyPressedD.value = false;
    } else if (keyEvent.code == 'KeyI') {
      keyPressedI.value = false;
    } else if (keyEvent.code == 'KeyJ') {
      keyPressedJ.value = false;
    } else if (keyEvent.code == 'KeyK') {
      keyPressedK.value = false;
    } else if (keyEvent.code == 'KeyL') {
      keyPressedL.value = false;
    } else if (keyEvent.code == 'Equal') {
      keyPressedPlus.value = false;
    } else if (keyEvent.code == 'Minus') {
      keyPressedMinus.value = false;
    } else if (keyEvent.code == 'ShiftLeft' || keyEvent.code == 'ShiftRight') {
      keyPressedShift.value = false;
    } else if (keyEvent.code == 'AltLeft' || keyEvent.code == 'AltRight') {
      keyPressedAlt.value = false;
    }
  });
  // Scroll-based rotation: map page scroll velocity to kaleidoscope spin
  lastWindowScrollY = window.scrollY;
  lastWindowScrollTime = performance.now();
  const onWindowScroll = () => {
    const now = performance.now();
    const dy = window.scrollY - lastWindowScrollY;
    const dt = Math.max(1, now - lastWindowScrollTime); // ms
    // dy/dt = px per ms; scale to a gentle rotation velocity impulse
    const velocityImpulse = (dy / dt) * SCROLL_ROTATION_SCALE;
    scopeRotationVel.value = clampRotationVelocity(scopeRotationVel.value + velocityImpulse);
    lastWindowScrollY = window.scrollY;
    lastWindowScrollTime = now;
  };
  window.addEventListener('scroll', onWindowScroll, { passive: true });
  onUnmounted(() => {
    window.removeEventListener('scroll', onWindowScroll);
  });
});

// Watch for changes to uploaded images
watch(() => props.uploadedImages, async (newImages, oldImages) => {
  // Fast-path: a single removal (avoids reloading & preserves active index)
  if (newImages && oldImages && oldImages.length - newImages.length === 1) {
    let removedIndex: number | null = null;
    let j = 0;
    for (let i = 0; i < oldImages.length; i += 1) {
      if (j >= newImages.length || oldImages[i] !== newImages[j]) {
        removedIndex = i;
        break;
      }
      j += 1;
    }
    if (removedIndex === null) {
      removedIndex = oldImages.length - 1;
    }

    // If we already removed locally as part of a dismiss gesture, just clear the pending marker.
    if (pendingOrbRemovalIndex.value !== null && removedIndex === pendingOrbRemovalIndex.value) {
      pendingOrbRemovalIndex.value = null;
      if (newImages.length > 0) {
        syncSlotIndices();
        orbTransitionProgress.value = 0.0;
        isOrbTransitioning.value = false;
        return;
      }
    }

    // Otherwise apply the same removal locally without reloading images.
    if (uploadedImageElements.value.length > 0) {
      removeUploadedImageAtIndex(removedIndex);
      return;
    }
  }

  if (newImages && newImages.length > 0) {
    await loadUploadedImages(newImages);
  } else {
    uploadedImageElements.value = [];
    activeImageIndex.value = 0;
    orbScrollOffset.value = 0;
    syncSlotIndices();
    orbTransitionProgress.value = 0.0;
    isOrbTransitioning.value = false;
    // Restart camera if no images are uploaded
    const camera = document.getElementById('camera') as HTMLVideoElement | null;
    if (camera && !cameraStream) {
      try {
        cameraStream = await navigator.mediaDevices.getUserMedia({video: { facingMode: { exact: 'environment'} }, audio: false});
        facingMode.value = cameraStream.getVideoTracks()[0]?.getSettings().facingMode ?? 'user';
      } catch {
        try {
          cameraStream = await navigator.mediaDevices.getUserMedia({video: true, audio: false});
          facingMode.value = cameraStream.getVideoTracks()[0]?.getSettings().facingMode ?? 'user';
        } catch (e2) {
          console.error('Failed to restart camera', e2);
        }
      }
      if (cameraStream && camera) {
        camera.srcObject = cameraStream;
        camera.play();
      }
    }
  }
}, { immediate: false });

</script>

<template>
  <!-- Full-viewport wrapper so #app has height and toolbar stays at bottom -->
  <div class="w-[100dvw] h-[100dvh] relative overflow-hidden">
    <div
      ref="interaction-layer"
      class="absolute inset-0 z-20 touch-none select-none"
    />
    <!-- Incoming orb (offscreen, moves into top or bottom small orb) -->
    <div
      class="absolute left-1/2 overflow-hidden rounded-full bg-neutral-800 z-0 w-[64vmin] h-[64vmin] min-w-[160px] min-h-[160px]"
      :style="getOrbStyle('incoming')"
      aria-hidden="true"
    >
      <canvas
        ref="display-canvas-incoming"
        class="block w-full h-full object-cover"
      />
    </div>
    <!-- Top orb (bottom hemisphere visible) -->
    <div
      class="absolute left-1/2 overflow-hidden rounded-full bg-neutral-800 z-0 w-[64vmin] h-[64vmin] min-w-[160px] min-h-[160px]"
      :style="getOrbStyle('top')"
      aria-hidden="true"
    >
      <canvas
        ref="display-canvas-top"
        class="block w-full h-full object-cover"
      />
    </div>

    <!-- Center orb (full) -->
    <div
      ref="center-orb"
      class="absolute left-1/2 overflow-hidden rounded-full bg-neutral-800 z-10 w-[64vmin] h-[64vmin] min-w-[160px] min-h-[160px]"
      :style="getCenterOrbStyle()"
    >
      <canvas
        ref="display-canvas-center"
        class="block w-full h-full object-cover"
      />
    </div>

    <!-- Bottom orb (top hemisphere visible) -->
    <div
      class="absolute left-1/2 overflow-hidden rounded-full bg-neutral-800 z-0 w-[64vmin] h-[64vmin] min-w-[160px] min-h-[160px]"
      :style="getOrbStyle('bottom')"
      aria-hidden="true"
    >
      <canvas
        ref="display-canvas-bottom"
        class="block w-full h-full object-cover"
      />
    </div>

    <!-- Hidden WebGL canvas used as source -->
    <canvas
      id="maincanvas"
      ref="canvas"
      class="absolute -left-[9999px] -top-[9999px] opacity-0 pointer-events-none"
    />
    <video
      id="camera"
      visible="False"
      style="width: 512px; height: 512px; display:none;"
      controls="true"
      playsinline
      crossorigin="anonymous"
    />
  </div>
</template>

<style scoped>
</style>
