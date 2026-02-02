<script setup lang="ts">
import {ref, onMounted, nextTick, useTemplateRef, watch} from 'vue';
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
const scopeSize = ref(1);
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
const orbTransitionStartTime = ref(0);
const ORB_TRANSITION_DURATION_MS = 450;
const orbScrollVelocity = ref(0);
const orbScrollProgress = ref(0);
const ORB_SCROLL_SPRING = 0.08;
const ORB_SCROLL_DAMPING = 0.85;
const ORB_SCROLL_IMPULSE = 0.0016;
const ORB_SWIPE_IMPULSE = 0.012;
const ORB_SCROLL_FAST_THRESHOLD = 0.08;
const maxRotationSpeed = 0.01; // Maximum rotation velocity
const maxScopeSizeVel = 0.12; // Maximum zoom velocity for physics follow-through
let cameraStream: MediaStream | null = null;

// Horizontal drag-to-dismiss (removes the active uploaded image)
const ORB_DISMISS_DURATION_MS = 200;
const orbDragTranslateX = ref(0);
const orbDragTranslateY = ref(0);
const orbDragScaleMultiplier = ref(1);
const isOrbDragging = ref(false);
const isOrbDismissing = ref(false);
let orbDragMouseStart: null | { x: number; y: number } = null;
let orbDragMouseActive = false;
let orbDragTouchActive = false;
let orbDragTouchLast: null | Point = null;
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

const startOrbTransition = (direction: 1 | -1) => {
  if (isOrbTransitioning.value) {
    return;
  }
  if (uploadedImageElements.value.length <= 1) {
    return;
  }
  orbTransitionDirection.value = direction;
  orbTransitionProgress.value = 0.0;
  orbTransitionStartTime.value = performance.now();
  isOrbTransitioning.value = true;
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

const getOrbStyle = (slot: 'top' | 'center' | 'bottom') => {
  const progress = isOrbTransitioning.value ? orbTransitionProgress.value : 0;
  const direction = orbTransitionDirection.value;

  let topPercent = slot === 'top' ? ORB_TOP_Y : slot === 'center' ? ORB_CENTER_Y : ORB_BOTTOM_Y;
  let scale = slot === 'center' ? 1 : ORB_SMALL_SCALE;

  if (progress > 0) {
    if (direction === 1) {
      if (slot === 'top') {
        topPercent = lerp(ORB_TOP_Y, ORB_OFFSCREEN_TOP, progress);
      } else if (slot === 'center') {
        topPercent = lerp(ORB_CENTER_Y, ORB_TOP_Y, progress);
        scale = lerp(1, ORB_SMALL_SCALE, progress);
      } else {
        topPercent = lerp(ORB_BOTTOM_Y, ORB_CENTER_Y, progress);
        scale = lerp(ORB_SMALL_SCALE, 1, progress);
      }
    } else {
      if (slot === 'bottom') {
        topPercent = lerp(ORB_BOTTOM_Y, ORB_OFFSCREEN_BOTTOM, progress);
      } else if (slot === 'center') {
        topPercent = lerp(ORB_CENTER_Y, ORB_BOTTOM_Y, progress);
        scale = lerp(1, ORB_SMALL_SCALE, progress);
      } else {
        topPercent = lerp(ORB_TOP_Y, ORB_CENTER_Y, progress);
        scale = lerp(ORB_SMALL_SCALE, 1, progress);
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

const SWIPE_DISTANCE_PX = 60;
const isSwipeGesture = (deltaX: number, deltaY: number): boolean => {
  return Math.abs(deltaY) >= SWIPE_DISTANCE_PX && Math.abs(deltaY) > Math.abs(deltaX);
};

const handleSwipeGesture = (deltaX: number, deltaY: number): boolean => {
  if (!isSwipeGesture(deltaX, deltaY)) {
    return false;
  }
  orbScrollVelocity.value += Math.max(-1, Math.min(1, deltaY / 200)) * ORB_SWIPE_IMPULSE;
  return true;
};

const isPointInsideCenterOrb = (p: Point): boolean => {
  const el = centerOrb.value as HTMLDivElement | undefined;
  if (!el) return false;
  const rect = el.getBoundingClientRect();
  const cx = rect.left + rect.width / 2;
  const cy = rect.top + rect.height / 2;
  const r = Math.min(rect.width, rect.height) / 2;
  return Math.hypot(p.clientX - cx, p.clientY - cy) <= r;
};

const getCenterOrbDismissThresholdPx = (): number => {
  const el = centerOrb.value as HTMLDivElement | undefined;
  if (!el) return 140;
  const rect = el.getBoundingClientRect();
  return Math.max(90, Math.min(220, rect.width * 0.28));
};

const getOrbDragArcOffsetY = (deltaX: number) => {
  if (typeof window === 'undefined') return 0;
  const el = centerOrb.value as HTMLDivElement | undefined;
  const viewportWidth = window.innerWidth || 1;
  const maxHorizontal = viewportWidth * 0.5;
  // From vertical center (~50vh) down to bottom (~100vh)
  const maxDownPx = window.innerHeight * 0.5;
  const norm = Math.min(1, Math.abs(deltaX) / Math.max(1, maxHorizontal));
  // Slightly eased curve
  const curved = norm * norm;
  return curved * maxDownPx;
};

const setOrbDragVisuals = (deltaX: number) => {
  orbDragTranslateX.value = deltaX;
  orbDragTranslateY.value = getOrbDragArcOffsetY(deltaX);
  // Allow the orb stack to get noticeably smaller as it approaches the corners
  const shrink = Math.min(0.7, Math.abs(deltaX) / 400);
  orbDragScaleMultiplier.value = Math.max(0.25, 1 - shrink);
};

const resetOrbDragVisuals = () => {
  orbDragTranslateX.value = 0;
  orbDragTranslateY.value = 0;
  orbDragScaleMultiplier.value = 1;
};

const removeUploadedImageAtIndex = (removeIndex: number) => {
  const total = uploadedImageElements.value.length;
  if (total <= 0) return;
  const wrappedRemoveIndex = getWrappedIndex(removeIndex, total);
  const wrappedActiveIndex = getWrappedIndex(activeImageIndex.value, total);

  uploadedImageElements.value.splice(wrappedRemoveIndex, 1);

  const newTotal = uploadedImageElements.value.length;
  if (newTotal <= 0) {
    activeImageIndex.value = 0;
    syncSlotIndices();
    orbTransitionProgress.value = 0.0;
    isOrbTransitioning.value = false;
    return;
  }

  if (wrappedRemoveIndex < wrappedActiveIndex) {
    activeImageIndex.value = Math.max(0, wrappedActiveIndex - 1);
  } else if (wrappedRemoveIndex === wrappedActiveIndex) {
    activeImageIndex.value = Math.min(wrappedActiveIndex, newTotal - 1);
  } else {
    activeImageIndex.value = Math.min(wrappedActiveIndex, newTotal - 1);
  }

  syncSlotIndices();
  orbTransitionProgress.value = 0.0;
  isOrbTransitioning.value = false;
};

const dismissActiveOrb = async (direction: 1 | -1, removeIndex: number) => {
  if (isOrbTransitioning.value || isOrbDismissing.value) return;
  if (uploadedImageElements.value.length <= 0) return;

  isOrbDismissing.value = true;
  // Ensure we animate out even if released near center
  orbDragTranslateX.value = direction * Math.max(window.innerWidth, 600);
  orbDragTranslateY.value = getOrbDragArcOffsetY(orbDragTranslateX.value);
  orbDragScaleMultiplier.value = 0;

  await new Promise((resolve) => setTimeout(resolve, ORB_DISMISS_DURATION_MS));

  // Remove locally immediately for responsiveness, and notify parent to update `uploadedImages`.
  pendingOrbRemovalIndex.value = removeIndex;
  removeUploadedImageAtIndex(removeIndex);
  emit('remove-uploaded-image', removeIndex);

  isOrbDismissing.value = false;
  resetOrbDragVisuals();
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
          
          // Motion blur based on rotation velocity - apply in texture coordinate space
          // Only apply motion blur when rotation is significant to save performance
          float motionBlurStrength = abs(rotationVelocity) * 8.0; // Increased multiplier for visibility
          motionBlurStrength = min(motionBlurStrength, 1.2); // Cap maximum blur
          
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

  function animate(){
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
      scopeRotationVel.value = clampRotationVelocity(props.scopeAutoRotationVelocity / 25);
      scopeRotation.value += scopeRotationVel.value;
    }
    if (touchOrigin1 === null && mousePrevPosition === null) {
      scopeRotation.value += scopeRotationVel.value;
      scopeRotationVel.value = clampRotationVelocity(scopeRotationVel.value * 0.99);
      scopeSizeVel.value *= 0.95;
      scopeSize.value = Math.max(0.5, Math.min(1, scopeSize.value * (1 + Math.min(scopeSizeVel.value, 0.99))));
    }
    scopeOffsetVel.value[0] *= 0.95;
    scopeOffsetVel.value[1] *= 0.95;

    if (isOrbTransitioning.value) {
      const now = performance.now();
      const elapsed = now - orbTransitionStartTime.value;
      orbTransitionProgress.value = Math.min(elapsed / ORB_TRANSITION_DURATION_MS, 1.0);
      if (orbTransitionProgress.value >= 1.0) {
        const totalImages = uploadedImageElements.value.length;
        if (totalImages > 0) {
          const delta = orbTransitionDirection.value === 1 ? 1 : -1;
          activeImageIndex.value = getWrappedIndex(activeImageIndex.value + delta, totalImages);
        }
        syncSlotIndices();
        isOrbTransitioning.value = false;
        orbTransitionProgress.value = 0.0;
        if (Math.abs(orbScrollVelocity.value) > ORB_SCROLL_FAST_THRESHOLD) {
          startOrbTransition(orbScrollVelocity.value > 0 ? 1 : -1);
        }
      }
    }

    if (!isOrbTransitioning.value) {
      orbScrollVelocity.value += -orbScrollProgress.value * ORB_SCROLL_SPRING;
      orbScrollProgress.value += orbScrollVelocity.value;
      orbScrollVelocity.value *= ORB_SCROLL_DAMPING;
      if (Math.abs(orbScrollVelocity.value) < 0.00001) {
        orbScrollVelocity.value = 0;
      }
      if (Math.abs(orbScrollProgress.value) >= 1) {
        const direction = orbScrollProgress.value > 0 ? 1 : -1;
        orbScrollProgress.value = 0;
        orbScrollVelocity.value *= 0.6;
        startOrbTransition(direction);
      }
    } else {
      orbScrollVelocity.value *= ORB_SCROLL_DAMPING;
    }

    const hasUploadedImages = uploadedImageElements.value.length > 0;
    const topImage = hasUploadedImages ? uploadedImageElements.value[topSlotIndex.value] : null;
    const centerImage = hasUploadedImages ? uploadedImageElements.value[centerSlotIndex.value] : null;
    const bottomImage = hasUploadedImages ? uploadedImageElements.value[bottomSlotIndex.value] : null;
    const fallbackSource = camera;

    gl.uniform1i(dataIsFacingUserBind, facingMode.value === 'user' ? 1 : 0);
    gl.uniform1i(scopeShapeBind, props.scopeShape);
    gl.uniform1f(scopeRotationBind, scopeRotation.value + scopeRotationOffset);
    gl.uniform1f(scopeSizeBind, scopeSize.value);
    gl.uniform2f(scopeOffsetBind, scopeOffset.value[0], scopeOffset.value[1]);
    gl.uniform1f(rotationVelocityBind, scopeRotationVel.value);
    gl.uniform2f(canvasDimensionsBind, canvasSize, canvasSize);

    drawOrbFrame(displayCanvasTop.value ?? null, topImage ?? fallbackSource, ORB_SCOPE_SCALE_MULTIPLIER);
    drawOrbFrame(displayCanvasBottom.value ?? null, bottomImage ?? fallbackSource, ORB_SCOPE_SCALE_MULTIPLIER);
    drawOrbFrame(displayCanvasCenter.value ?? null, centerImage ?? fallbackSource, 1);

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
      orbDragTouchLast = touch;
      setOrbDragVisuals(touch.clientX - touchOrigin1.clientX);
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
    const pos = { x: mouseEvent.clientX, y: mouseEvent.clientY };
    const shouldOrbDrag =
      uploadedImageElements.value.length > 0 &&
      !isOrbTransitioning.value &&
      !isOrbDismissing.value &&
      isPointInsideCenterOrb(mouseEvent);
    if (shouldOrbDrag) {
      orbDragMouseActive = true;
      orbDragMouseStart = pos;
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
      setOrbDragVisuals(mouseEvent.clientX - orbDragMouseStart.x);
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
    isUserPressing.value = false;
    mousePrevPosition = null;
    mouseStartPosition = null;
  });

  document.addEventListener('wheel', (wheelEvent) => {
    wheelEvent.preventDefault();
    if (Math.abs(wheelEvent.deltaY) < 4) {
      return;
    }
    const clampedDelta = Math.max(-80, Math.min(80, wheelEvent.deltaY));
    orbScrollVelocity.value += clampedDelta * ORB_SCROLL_IMPULSE;
  }, { passive: false });

  interactionElement.addEventListener('touchstart', touchStartCallback);
  interactionElement.addEventListener('touchmove', touchMoveCallback);
  interactionElement.addEventListener('touchend', touchEndCallback);
  interactionElement.addEventListener('touchcancel', touchCancelCallback);

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
    <div ref="interaction-layer" class="absolute inset-0 z-20 touch-none select-none" />
    <!-- Top orb (bottom hemisphere visible) -->
    <div
      class="absolute left-1/2 overflow-hidden rounded-full bg-[#EAEAE8] z-0 w-[80vmin] h-[80vmin] min-w-[200px] min-h-[200px]"
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
      class="absolute left-1/2 overflow-hidden rounded-full bg-[#EAEAE8] z-10 w-[80vmin] h-[80vmin] min-w-[200px] min-h-[200px] will-change-transform"
      :class="isOrbDragging ? '' : 'transition-transform duration-200 ease-out'"
      :style="getOrbStyle('center')"
    >
      <canvas
        ref="display-canvas-center"
        class="block w-full h-full object-cover"
      />
    </div>

    <!-- Bottom orb (top hemisphere visible) -->
    <div
      class="absolute left-1/2 overflow-hidden rounded-full bg-[#EAEAE8] z-0 w-[80vmin] h-[80vmin] min-w-[200px] min-h-[200px]"
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
