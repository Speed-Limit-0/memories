<script setup lang="ts">
import {ref, onMounted, nextTick, useTemplateRef, watch, onUnmounted} from 'vue';
import { ScopeShape } from '../scopeShape.ts';

const props = defineProps<{
  scopeShape: ScopeShape,
  scopeAutoRotationVelocity: number
  saveNextFrame: boolean
  uploadedImages?: string[]
}>();

const emit = defineEmits(['save-frame', 'upload-click', 'remove-uploaded-image', 'gallery-view-change', 'active-orbs-change']);

const CLICK_MOVE_THRESHOLD_PX = 10;


const facingMode = ref('unknown');
const cameraZoom = ref(1);
// Independent rotations for each orb (legacy refs kept for compatibility until full physics migration)
const scopeSize = ref(0.8);
const scopeOffset = ref([0.0, 0.0]);
const scopeSizeVel = ref(0.0);
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
const interactionLayer = useTemplateRef('interaction-layer');
const uploadedImageElements = ref([] as HTMLImageElement[]);
const activeImageIndex = ref(0);
// Removed: topSlotIndex, centerSlotIndex, bottomSlotIndex, orbTransitionProgress, etc. as we move to new system


interface OrbState {
  id: string;
  img: HTMLImageElement;
  x: number; // visual x-offset (for dismiss/drag)
  y: number; // visual position in pixels relative to center
  vy: number;
  scale: number;
  rotation: number;
  rotationVel: number;
  texture: WebGLTexture | null;
  galleryId: string; // Link to persistent gallery item
}

interface GalleryItem {
  id: string;
  src: string;
  img?: HTMLImageElement; // Keep ref for baking
  thumbnailSrc?: string; // Baked kaleidoscope preview
  originalSrc: string; // Keep original reference
  status: 'active' | 'left' | 'right';
}

// Global helper to bake thumbnails (assigned in main)
let bakeKaleidoscopeThumbnail: ((img: HTMLImageElement, zoomLevel?: 'in' | 'out') => string) | null = null;

// Global GL context for shared access
let gl: WebGLRenderingContext | null = null;


const orbs = ref<OrbState[]>([]);
const galleryItems = ref<GalleryItem[]>([]);
// Global scroll anchor (target position for index 0)
// We treat "1 unit" of scroll as "one orb height + gap"
const scrollAnchor = ref(0);
const scrollAnchorVel = ref(0);
const pendingScrollToGallery = ref(false);

// Physics Constants
// Physics Constants
// Physics Constants
const ORB_HEIGHT_PX = ref(0); 
const ORB_SPRING_STIFFNESS = 120;
const ORB_SPRING_DAMPING = 20; // Critical damping ~ sqrt(4*k) -> sqrt(480) ~ 22. So 20 is slightly underdamped.
const ORB_GAP_PX = ref(0); 
const ORB_SPACING = ref(0);

// Target Locking Physics
const scrollTarget = ref<number | null>(null);
const WHEEL_LOCK_TIMEOUT_MS = 150; // Delay after wheel stops to lock target

const updateLayout = () => {
    // The requirement: "half hemisphere of the smaller orbs above/below should be visible"
    // This implies that the spacing between orb centers should be half the viewport height.
    // So if the center orb is at 0, the next one is at +window.innerHeight/2.
    // This places the center of the next orb exactly at the bottom edge of the screen.
    
    // We update the reactive constants
    const vh = window.innerHeight;
    const spacing = vh * 0.5; // distance between centers
    
    ORB_SPACING.value = spacing;
    
    // ORB_HEIGHT_PX is used for visual sizing logic in some places, 
    // keep consistent with the visual CSS (64vmin)
    const vmin = Math.min(window.innerWidth, window.innerHeight);
    const visualSize = Math.max(160, vmin * 0.64);
    ORB_HEIGHT_PX.value = visualSize;
    
    // Gap is just derived
    ORB_GAP_PX.value = ORB_SPACING.value - ORB_HEIGHT_PX.value;
};

// Mapping: scrollAnchor = 0 -> orb[0] is at center
// scrollAnchor = 1 -> orb[1] is at center (orb[0] moves up)
// Physics & Debug State
// Physics tuning parameters
const debugMaxImpulse = ref(4.0);
const debugSpringTension = ref(300.0);
const debugSpringFriction = ref(48.0);
const debugFlickMultiplier = ref(1.0);
const debugSnapbackThreshold = ref(1.0);

// Simplified impulse mapping parameters (single mapping for wheel/trackpad/touch)
const ORB_WHEEL_DELTA_MAX = 120; // clamp reference for raw wheel delta
const ORB_IMPULSE_EXP = 1.8; // nonlinear exponent (>1 makes large deltas grow faster)
const ORB_TRACKPAD_SCALE = 0.9; // slight device scale for trackpad
const maxRotationSpeed = 1; // Maximum rotation velocity
const maxScopeSizeVel = 0.12; // Maximum zoom velocity for physics follow-through
// Scroll -> rotation mapping: scale factor applied to scroll velocity (px/ms) to rotation velocity
const SCROLL_ROTATION_SCALE = 0.003;
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
let orbDragMouseStart: null | { x: number; y: number } = null;
let orbDragMouseActive = false;
let orbDragTouchActive = false;
let orbDragTouchLast: null | Point = null;
let orbDragLastX: null | number = null;
const pendingOrbRemovalIndex = ref<number | null>(null);

// New Interaction State
const activeOrbDragIndex = ref<number | null>(null);
const isVerticalDrag = ref(false);
const dragStartY = ref(0);
const dragStartScrollOffset = ref(0);
const dragLastY = ref(0);
const dragLastTime = ref(0);
const dragVelocity = ref(0); // in progress units per ms
const hasDragMoved = ref(false); 

const clampRotationVelocity = (velocity: number): number => {
  return Math.max(-maxRotationSpeed, Math.min(maxRotationSpeed, velocity));
};
const clampScopeSizeVelocity = (velocity: number): number => {
  return Math.max(-maxScopeSizeVel, Math.min(maxScopeSizeVel, velocity));
};
let texture1: WebGLTexture | null = null;
let texture2: WebGLTexture | null = null;

// Helper to create texture from image
const createTextureFromImage = (img: HTMLImageElement): WebGLTexture | null => {
  if (!gl) return null;
  const tex = gl.createTexture();
  if (!tex) return null;
  
  gl.bindTexture(gl.TEXTURE_2D, tex);
  // Upload the image into the texture.
  gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, img);
  
  // Set the structural parameters.
  gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, true);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
  
  // Unbind
  gl.bindTexture(gl.TEXTURE_2D, null);
  return tex;
};

const canvasRefs = ref<Record<string, HTMLCanvasElement>>({});
const setCanvasRef = (el: any, id: string) => {
  if (el) canvasRefs.value[id] = el as HTMLCanvasElement;
};

const gridCanvasRefs = ref<Record<string, HTMLCanvasElement>>({});

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




// ----------------------------------------------------------------------
// Legacy scroll logic removed (applyOrbScrollDelta, startOrbSnap, etc.)
// ----------------------------------------------------------------------

// Convert raw input delta (wheel delta or touch delta) into a sensible impulse.
const computeImpulseFromDelta = (rawDelta: number, isTrackpad: boolean) => {
  const sign = Math.sign(rawDelta) || 1;
  const absClamped = Math.min(ORB_WHEEL_DELTA_MAX, Math.abs(rawDelta));
  const normalized = absClamped / ORB_WHEEL_DELTA_MAX; // 0..1
  const scaled = Math.pow(normalized, ORB_IMPULSE_EXP);
  const base = scaled * debugMaxImpulse.value;
  const deviceScale = isTrackpad ? ORB_TRACKPAD_SCALE : 1;
  return sign * base * deviceScale;
};




const getOrbStyle = (orb: OrbState) => {
  // orb.y is pixels from center (0)
  // We center the orb at 50% of container, then translate by orb.y
  
  // Scale effect: Grow slightly when near center?
  // We can compute this dynamically in the loop or here.
  // Let's use the current "distance from 0" logic for scale visual
  const dist = Math.abs(orb.y);
  // Scale ends at 0.5 (half size) when distance is >= ORB_SPACING
  // Scale is 1.0 at distance 0
  const distRatio = Math.min(1.0, dist / ORB_SPACING.value);
  const scaleFactor = 1.0 - (0.5 * distRatio); // properties: at 0 -> 1.0. at 1 -> 0.5.
  
  const visualScale = orb.scale * scaleFactor;
  
  // Center Orbit Drag/Dismiss logic (only if active?)
  // If we want to support dragging ANY orb, we can check IDs.
  // For now, let's just apply the transform.
  
  return {
     top: '50%',
     transform: `translate(-50%, -50%) translate(${orb.x}px, ${orb.y}px) scale(${visualScale})`,
     zIndex: Math.round(100 - dist / 10), // closer -> higher z-index
  };
};

const getOrbScopeScale = () => {
   // Legacy shim for drawOrbFrame to compile
   return 1.0; 
};



const getReadySource = (
  primary: HTMLImageElement | null,
  fallback: HTMLCanvasElement
): HTMLImageElement | HTMLCanvasElement => {
  if (primary && primary.complete && primary.naturalWidth > 0) {
    return primary;
  }
  return fallback;
};


const loadUploadedImages = async (imageSrcs: string[]) => {
  if (imageSrcs.length > 0) {
    const loadedOrbs: OrbState[] = [];
    let index = 0;
    for (const src of imageSrcs) {
      const img = new Image();
      img.crossOrigin = 'anonymous';
      img.src = src;
      await new Promise((resolve, reject) => {
        img.onload = resolve;
        img.onerror = reject;
      });
      const resizedImg = await resizeImage(img);
      
      // Initialize orb state
      // Initial position: stacked vertically based on index
      // Target position will be calculated in the physics loop
      // Reconcile with gallery items
      let galleryId = '';
      const existingItem = galleryItems.value.find(item => item.src === src && item.status === 'active' && !loadedOrbs.some(o => o.galleryId === item.id));
      
      if (existingItem) {
        galleryId = existingItem.id;
      } else {
        galleryId = `gallery-${Date.now()}-${index}`;
        
        // Generate baked thumbnail if possible (normal size for active items)
        let thumb = undefined;
        if (bakeKaleidoscopeThumbnail) {
             try {
                thumb = bakeKaleidoscopeThumbnail(resizedImg, 'in');
             } catch (e) {
                console.error("Failed to bake thumbnail", e);
             }
        }
        
        galleryItems.value.push({
          id: galleryId,
          src: src,
          img: resizedImg, 
          thumbnailSrc: thumb,
          originalSrc: src,
          status: 'active'
        });
      }

      loadedOrbs.push({
        id: `orb-${Date.now()}-${index}`,
        img: resizedImg,
        x: 0,
        y: index * ORB_SPACING.value, // Initial placement
        vy: 0,
        scale: 1,
        rotation: 0,
        rotationVel: 0,
        texture: createTextureFromImage(resizedImg),
        galleryId: galleryId
      });
      index++;
    }
    
    // Update State
    uploadedImageElements.value = loadedOrbs.map(o => o.img); // Keep for legacy compat references if needed
    orbs.value = loadedOrbs;
    activeImageIndex.value = 0;
    
    // Reset Scroll Anchor to 0
    scrollAnchor.value = 0;
    scrollAnchorVel.value = 0;
    
    facingMode.value = 'environment';
  } else {
    uploadedImageElements.value = [];
    orbs.value = [];
    activeImageIndex.value = 0;
    scrollAnchor.value = 0;
  }
};

const resolveScrollTarget = () => {
    const current = scrollAnchor.value;
    const velocity = scrollAnchorVel.value;
    const mag = Math.abs(velocity);
    const sign = Math.sign(velocity) || 1;
    
    // Always calculate a target if we have velocity
    if (mag > 0.01) {
        // SNAPBACK ZONE: If velocity is too low, don't leave the current orb.
        // This creates the "magnetic" pull feeling.
        if (mag < debugSnapbackThreshold.value) {
             scrollTarget.value = Math.max(0, Math.min(orbs.value.length + 1, Math.round(current)));
             return; 
        }

        // Fling / Scroll Command: Just go to the next/nearest orb in direction of travel
        // We ignore debugBreakthroughVel and steps logic to ensure we only go to the "next one"
        const steps = 1;
        
        // Sticky Base: The "next" orb in direction of travel
        // If sign > 0 (scrolling down), base is floor(current)
        // If sign < 0 (scrolling up), base is ceil(current)
        const base = sign > 0 ? Math.floor(current) : Math.ceil(current);
        let target = base + steps * sign;
        
        // Skip position orbs.length in both directions - go directly to/from gallery
        if (target === orbs.value.length) {
            // Scrolling down from last orb -> skip to gallery
            target = sign > 0 ? orbs.value.length + 1 : orbs.value.length - 1;
        }
        
        scrollTarget.value = Math.max(0, Math.min(orbs.value.length + 1, target));

        // Velocity dampening: limit velocity so the spring settles quickly without large overshoot
        if (Math.abs(scrollAnchorVel.value) > 6.0) {
            scrollAnchorVel.value = sign * 6.0;
        }
    } else {
        // If almost stopped, snap to absolute nearest
        scrollTarget.value = Math.max(0, Math.min(orbs.value.length + 1, Math.round(current)));
    }
};

async function main(canvasElement: HTMLCanvasElement) {
  // No webcam: use a small placeholder canvas as fallback when orb image isn't ready
  const fallbackCanvas = document.createElement('canvas');
  fallbackCanvas.width = 2;
  fallbackCanvas.height = 2;
  const ctx = fallbackCanvas.getContext('2d');
  if (ctx) {
    ctx.fillStyle = '#1a1a1a';
    ctx.fillRect(0, 0, 2, 2);
  }

  // If images are uploaded, create image elements for them
  if (props.uploadedImages && props.uploadedImages.length > 0) {
    await loadUploadedImages(props.uploadedImages);
  }

  // Canvas with WebGL context (element passed from template ref so it exists when mounted)
  // Optimize: match canvas size to the visual orb size (64vmin), not the restart of the screen.
  // Also cap pixel ratio to 2.0 to avoid excessive overhead on high-DPI mobile screens.
  const dpr = Math.min(window.devicePixelRatio || 1, 2.0);
  const vmin = Math.min(window.innerWidth, window.innerHeight);
  // 64vmin is the CSS size. We add a bit of buffer for safety/quality, but not full screen.
  // 0.64 * vmin * dpr. 
  // Let's cap it at 1024 to be safe, but usually it will be smaller on mobile.
  // e.g. iPhone width 400 * 0.64 * 3 = 768. 
  // Full screen was ~3000px.
  const orbVisualSize = Math.round(vmin * 0.64 * dpr);
  const canvasSize = Math.max(800, orbVisualSize); // minimal quality baseline 800
  
  gl = canvasElement.getContext('webgl', { alpha: false, antialias: false, depth: false })!; // Optimize context attributes
  if (!gl) {
    console.error('WebGL not supported');
    return;
  }
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
    if (!gl) return;
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
    imageSource: HTMLImageElement | HTMLCanvasElement,
    scopeScaleMultiplier: number,
    cachedTexture: WebGLTexture | null = null
  ) => {
    const imageWidth = imageSource.width;
    const imageHeight = imageSource.height;
    if (imageWidth <= 0 || imageHeight <= 0) {
      return;
    }

    if (!gl) return;
    gl.activeTexture(gl.TEXTURE0);
    gl.bindTexture(gl.TEXTURE_2D, texture1); // Bind unit 0
    
    // Use cached texture if available
    if (cachedTexture) {
         // If we have a cached texture, we must bind IT to the active texture unit.
         // Wait, texture1 is the unit 0 texture...
         // Actually, we should bind cachedTexture INSTEAD of texture1 if it exists.
         gl.bindTexture(gl.TEXTURE_2D, cachedTexture);
    } else {
         gl.bindTexture(gl.TEXTURE_2D, texture1);
         gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, imageSource);
    }

    gl.uniform2f(dataDimensionsBind, imageWidth, imageHeight);
    gl.uniform2f(dataDimensions2Bind, imageWidth, imageHeight);
    gl.uniform1f(dataZoomBind, cameraZoom.value);
    gl.uniform1f(scopeSizeBind, scopeSize.value * scopeScaleMultiplier);
    if (transitionProgressBind) {
      gl.uniform1f(transitionProgressBind, 0.0);
    }
    // Draw call
    gl.drawArrays(gl.TRIANGLES, 0, 6);
    
    // If displayCanvas provided, copy to it. If null, we just drew to the main framebuffer (gl context).
    if (displayCanvas) {
        renderToDisplayCanvas(displayCanvas, canvasElement);
    }
  };

  // Implement the baker
  bakeKaleidoscopeThumbnail = (img: HTMLImageElement, zoomLevel: 'in' | 'out' = 'out') => {
      if (!gl) return '';
      // 1. Temporarily bind texture
      const tex = gl.createTexture();
      setupTexture(tex);
      gl.activeTexture(gl.TEXTURE0); // Use unit 0
      gl.bindTexture(gl.TEXTURE_2D, tex);
      gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, img);
      
      // 2. Render to main canvas (hidden)
      // Use standard settings for uniformity in gallery
      // Zoom levels: 'out' = LARGER scopeSize = fewer reflections (clearer), 'in' = normal scopeSize
      const baseScopeSize = scopeSize.value;
      const zoomedScopeSize = zoomLevel === 'out' ? baseScopeSize * 2.0 : baseScopeSize;
      
      gl.uniform2f(dataDimensionsBind, img.width, img.height);
      gl.uniform2f(dataDimensions2Bind, img.width, img.height);
      gl.uniform1f(dataZoomBind, 1.0); // Reset zoom for thumbnail
      gl.uniform1f(scopeSizeBind, zoomedScopeSize);
      gl.uniform1f(scopeRotationBind, 0.0); // Standardize rotation
      gl.uniform1f(transitionProgressBind, 0.0);
      
      gl.drawArrays(gl.TRIANGLES, 0, 6);
      
      // 3. Capture and resize
      // The main canvas might be huge. We want a small thumb.
      const thumbSize = 200;
      const tempCanvas = document.createElement('canvas');
      tempCanvas.width = thumbSize;
      tempCanvas.height = thumbSize;
      const tCtx = tempCanvas.getContext('2d');
      if (tCtx) {
          tCtx.drawImage(canvasElement, 0, 0, canvasElement.width, canvasElement.height, 0, 0, thumbSize, thumbSize);
           // Cleanup
           gl.deleteTexture(tex);
           // Resizing to small prevents massive base64 strings
           return tempCanvas.toDataURL('image/jpeg', 0.8);
       }
       
       gl.deleteTexture(tex);
       return canvasElement.toDataURL('image/jpeg', 0.8);
  };
  
  // Bake any pending items (from initial load)
  galleryItems.value.forEach(item => {
     if (!item.thumbnailSrc && item.img) {
         try {
            item.thumbnailSrc = bakeKaleidoscopeThumbnail!(item.img, 'in');
         } catch (e) {
            console.error("Failed to bake pending thumbnail", e);
         }
     }
  });


  // setCanvasRef definition removed from here (moved to top level)

  let lastAnimateTime = performance.now();
  
  function animate(){
    const now = performance.now();
    const dt = Math.min(0.032, (now - lastAnimateTime) / 1000); // sec
    lastAnimateTime = now;

    // 1. Scroll Anchor Physics

    if (!isUserPressing.value) {
       // Bounds logic handled by target clamping in resolveScrollTarget
       // Here we just spring towards scrollTarget if it exists
       
       if (scrollTarget.value !== null) {
           const dist = scrollTarget.value - scrollAnchor.value;
           const force = dist * debugSpringTension.value - scrollAnchorVel.value * debugSpringFriction.value;
           scrollAnchorVel.value += force * dt;
       } else {
           // We are in a "floating" state (e.g. wheel is active but hasn't timed out), 
           // just apply some friction so we don't drift forever
           scrollAnchorVel.value *= Math.exp(-3.0 * dt);
       }
       
       // Update position
       scrollAnchor.value += scrollAnchorVel.value * dt;

    } else {
       // User is pressing -> clear target
       scrollTarget.value = null;
    }

    // 2. Orb Physics & Rendering (Moved Rotation Physics here)
    const visibleOrbs: OrbState[] = [];
    
    // Auto-rotation base velocity
    const autoRotVel = props.scopeAutoRotationVelocity !== 0 
        ? clampRotationVelocity(props.scopeAutoRotationVelocity / 25) 
        : 0;

    for (let i = 0; i < orbs.value.length; i++) {
        const orb = orbs.value[i];
        
        // Target Y Position
        const targetY = (i - scrollAnchor.value) * ORB_SPACING.value;
        
        // Spring Force for Position
        const dist = targetY - orb.y;
        const springF = dist * ORB_SPRING_STIFFNESS;
        const dampingF = -orb.vy * ORB_SPRING_DAMPING;
        const totalF = springF + dampingF;
        
        orb.vy += totalF * dt;
        orb.y += orb.vy * dt;
        
        // --- Independent Rotation Physics ---
        // Apply auto-rotation driven velocity if set, otherwise just decay
        if (autoRotVel !== 0) {
           // Blend active velocity towards auto-velocity? 
           // Or just set it if it's the dominant force?
           // Original logic was: set it, then decay. 
           // To allow manual spin to override, we only apply auto if velocity is small?
           // Or just treat auto as a force?
           // Let's adopt the "set and decay" pattern from original checks but per orb, 
           // allowing manual impulses (which add large velocity) to temporarily override.
           
           // If user isn't actively spinning this orb (velocity is low/decayed), push it.
           // This is a heuristic.
           if (Math.abs(orb.rotationVel) < Math.abs(autoRotVel)) {
              orb.rotationVel = autoRotVel;
           }
        }
        
        orb.rotation += orb.rotationVel;
        orb.rotationVel *= 0.98; // Angular Drag
        
        // Render if visible (gross culling)
        if (Math.abs(orb.y) < window.innerHeight) { 
             visibleOrbs.push(orb);
        }
    }
    
    // 3. Render Cycle
    const imageFallback = fallbackCanvas;

    // Shared uniforms
    if (!gl) return;
    gl.uniform1i(dataIsFacingUserBind, facingMode.value === 'user' ? 1 : 0);
    gl.uniform1i(scopeShapeBind, props.scopeShape);
    gl.uniform2f(scopeOffsetBind, scopeOffset.value[0], scopeOffset.value[1]);
    gl.uniform2f(canvasDimensionsBind, canvasSize, canvasSize);

    visibleOrbs.forEach(orb => {
        const source = getReadySource(orb.img, imageFallback);
        const displayCanvas = canvasRefs.value[orb.id];
        
        if (displayCanvas && gl) {
             gl.uniform1f(scopeRotationBind, orb.rotation);
             gl.uniform1f(rotationVelocityBind, orb.rotationVel);
             
             // Scale effect: Grow slightly when near center?
             const dist = Math.abs(orb.y);
             const proximity = Math.max(0, 1 - dist / 400); // 0..1
             const scaleEffect = 1 + proximity * 0.2;
             
             drawOrbFrame(displayCanvas, source ?? imageFallback, getOrbScopeScale() * scaleEffect, orb.texture); 
        }
    });

    // 4. Grid Gallery Rendering (If visible)
    // Check if we are physically near the bottom
    // We render grid if scrollAnchor is nearing the end
    // Show grid if within ~2 screens of bottom?
    // Grid opacity logic in template: 1 - Math.abs((length - scroll) * 0.5)
    // So visible when abs diff < 2.
    if (Math.abs(orbs.value.length - scrollAnchor.value) < 3.0) {
        orbs.value.forEach(orb => {
            const gridCanvas = gridCanvasRefs.value[orb.id];
            if (gridCanvas) {
                const source = getReadySource(orb.img, imageFallback);
                if (gl) {
                  gl.uniform1f(scopeRotationBind, orb.rotation);
                  gl.uniform1f(rotationVelocityBind, orb.rotationVel);
                  // No extra scale effect for grid items
                  drawOrbFrame(gridCanvas, source ?? imageFallback, 1.0, orb.texture);
                }
            }
        });
    }

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

const getOrbIndexAtPoint = (p: {x: number, y: number}) => {
   // Only check visible orbs
   // Hit test: circle collision
   // We need to know where each orb is VISUALLY.
   // The 'orb.y' property is relative to the center.
   const cx = window.innerWidth / 2;
   const cy = window.innerHeight / 2;
   // Approx size (from CSS 64vmin, min 160px)
   const vmin = Math.min(window.innerWidth, window.innerHeight);
   const radius = Math.max(160, vmin * 0.64) / 2;
   
   for (let i = 0; i < orbs.value.length; i++) {
      const orb = orbs.value[i];
      const orbScreenY = cy + orb.y;
      const orbScreenX = cx + orb.x; // usually 0 unless dismissing
      const dx = p.x - orbScreenX;
      const dy = p.y - orbScreenY;
      if (Math.hypot(dx, dy) <= radius) {
         return i;
      }
   }
   return -1;
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
// Set drag visuals while dragging
const setOrbDragVisuals = (deltaX: number) => {
  if (activeOrbDragIndex.value !== null && activeOrbDragIndex.value >= 0 && activeOrbDragIndex.value < orbs.value.length) {
      orbs.value[activeOrbDragIndex.value].x = deltaX;
  }
};

const getCenterOrbDismissThresholdPx = (): number => {
  return Math.max(100, Math.round(Math.min(window.innerWidth, window.innerHeight) * 0.30));
};

const bounceBackOrb = async (index: number) => {
  if (index < 0 || index >= orbs.value.length) return;
  const orb = orbs.value[index];
  if (!orb) return;

  const startX = orb.x;
  const targetX = 0;
  const startTime = performance.now();
  const duration = 400;

  while (true) {
     const now = performance.now();
     const p = Math.min(1, (now - startTime) / duration);
     const ease = 1 - Math.pow(1 - p, 5); 
     orb.x = startX + (targetX - startX) * ease;
     if (p >= 1) break;
     await new Promise(r => requestAnimationFrame(r));
  }
  orb.x = 0;
};

const dismissActiveOrb = async (direction: 1 | -1, removeIndex: number) => {
  if (removeIndex < 0 || removeIndex >= orbs.value.length) return;
  const orb = orbs.value[removeIndex];
  if (!orb) return;

  isOrbDismissing.value = true;
  
  // Animate X out
  const startX = orb.x;
  const targetX = direction * window.innerWidth;
  const startTime = performance.now();
  const duration = ORB_DISMISS_DURATION_MS;
  
  // Simple animation loop (blocking)
  while (true) {
     const now = performance.now();
     const p = Math.min(1, (now - startTime) / duration);
     // Ease out
     const ease = 1 - Math.pow(1 - p, 3);
     orb.x = startX + (targetX - startX) * ease;
     if (p >= 1) break;
     await new Promise(r => requestAnimationFrame(r));
  }
  
  // Update Gallery Status
  // Find the gallery item
  const galleryItem = galleryItems.value.find(item => item.id === orb.galleryId);
  if (galleryItem) {
    galleryItem.status = direction === 1 ? 'right' : 'left';
    
    // Rebake thumbnail for both swipes (larger scope size for clearer view)
    if (galleryItem.img && bakeKaleidoscopeThumbnail) {
      try {
        galleryItem.thumbnailSrc = bakeKaleidoscopeThumbnail(galleryItem.img, 'out');
      } catch (e) {
        console.error('Failed to rebake thumbnail for swipe', e);
      }
    }
  }

  // Remove
  pendingOrbRemovalIndex.value = removeIndex;
  removeUploadedImageAtIndex(removeIndex);
  emit('remove-uploaded-image', removeIndex);
  
  isOrbDismissing.value = false;
  
  // The physics loop will automatically pull the next orbs up to fill the gap.
};

const removeUploadedImageAtIndex = (index: number) => {
  if (index < 0 || index >= uploadedImageElements.value.length) return;
  // Remove from arrays
  uploadedImageElements.value.splice(index, 1);
  if (index < orbs.value.length) {
     const removedOrb = orbs.value[index];
     if (removedOrb.texture && gl) {
        gl.deleteTexture(removedOrb.texture);
     }
     orbs.value.splice(index, 1);
  }

  // Adjust scroll position if we removed an item *above* our current view
  // This keeps the "current" item in view (which has now shifted index by -1)
  if (scrollAnchor.value > index) {
      scrollAnchor.value = Math.max(0, scrollAnchor.value - 1);
  }
  
  // Also adjust the target if it exists, so we don't snap back to the "old" index 
  // (which is now the next item)
  if (scrollTarget.value !== null && scrollTarget.value > index) {
      scrollTarget.value = Math.max(0, scrollTarget.value - 1);
  }
  
  if (activeImageIndex.value >= uploadedImageElements.value.length) {
    activeImageIndex.value = Math.max(0, uploadedImageElements.value.length - 1);
  }

  // Scroll to gallery is handled by the watcher on orbs.value.length
  // No syncSlotIndices needed, physics handles it
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
    
    // Initialize new drag state
    isVerticalDrag.value = false;
    hasDragMoved.value = false;
    dragStartY.value = touch.clientY;
    dragVelocity.value = 0;
    
    dragStartScrollOffset.value = scrollAnchor.value;
    
    // Check hit test
    const targetOrbIndex = getOrbIndexAtPoint({x: touch.clientX, y: touch.clientY});
    const shouldOrbDrag =
      uploadedImageElements.value.length > 0 &&
      !isOrbDismissing.value &&
      targetOrbIndex !== -1;

    if (shouldOrbDrag) {
      activeOrbDragIndex.value = targetOrbIndex;
      orbDragTouchActive = true;
      orbDragTouchLast = touch;
      orbDragLastX = touch.clientX;
      isOrbDragging.value = true;
      orbDragTranslateX.value = 0;
    } else {
      activeOrbDragIndex.value = null;
      orbDragTouchActive = false;
      orbDragTouchLast = null;
      isOrbDragging.value = false;
    }
    touchId1 = touch.identifier;
    touchPrev1 = touch;
    touchOrigin1 = touch;
    touchPrevTime = new Date().getTime();
    isUserPressing.value = true;
    scrollAnchorVel.value = 0; // Stop existing scroll momentum on touch down
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
    
    const now = performance.now();
    const dt = Math.max(1, now - dragLastTime.value);
    const dy = touch.clientY - dragLastY.value;
    
    // Unified Drag Logic (Vertical Scroll + Horizontal Move)

    // 1. Vertical Drag (Scroll)
    const totalDragY = touch.clientY - dragStartY.value;
    const progressDelta = -totalDragY / ORB_SPACING.value; // Up drag (negative Y) -> Positive scroll
    scrollAnchor.value = dragStartScrollOffset.value + progressDelta;
    
    // Update velocity for momentum (using immediate dy for responsiveness)
    scrollAnchorVel.value = (-dy / ORB_SPACING.value) / (dt / 1000); 

    // 2. Horizontal Drag (Rotation / Dismiss) - Only if dragging a specific orb
    if (orbDragTouchActive && activeOrbDragIndex.value !== null) {
      const index = activeOrbDragIndex.value;
      const orb = orbs.value[index];
      if (orb) {
          // Rotate kaleidoscope incrementally based on horizontal drag delta
          const prevX = orbDragLastX ?? touch.clientX;
          const dx = touch.clientX - prevX;
          const dtMove = Math.max(1, now - touchPrevTime);
          
          orb.rotation += dx / 300;
          orb.rotationVel = clampRotationVelocity((dx / dtMove) * 0.02);
          
          orbDragLastX = touch.clientX;
          orbDragTouchLast = touch;
          setOrbDragVisuals(touch.clientX - touchOrigin1.clientX);
      }
    }
    
    hasDragMoved.value = true; // Mark as moved so we don't treat as a static click later
    
    dragLastY.value = touch.clientY;
    dragLastTime.value = now;
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
    
    // Reset drag tracking for the new "single" touch
    isVerticalDrag.value = false;
    hasDragMoved.value = false;
    dragStartY.value = remaining.clientY;
    dragStartScrollOffset.value = scrollAnchor.value;
  }
  if (len === 0) {

    // Both momentum and dismiss checks happen now

    // Horizontal Dismiss / Bounce Logic
    if (orbDragTouchActive && touchOrigin1 !== null) {
      const last = orbDragTouchLast ?? (touchId1 !== null ? getTouchById(event.changedTouches, touchId1) : null);
      const deltaX = last ? last.clientX - touchOrigin1.clientX : orbDragTranslateX.value;
      const threshold = getCenterOrbDismissThresholdPx();
      const direction: 1 | -1 = deltaX >= 0 ? 1 : -1;
      
      const draggedIndex = activeOrbDragIndex.value ?? -1;
      activeOrbDragIndex.value = null;

      orbDragTouchActive = false;
      orbDragTouchLast = null;
      isOrbDragging.value = false;

      if (Math.abs(deltaX) >= threshold && uploadedImageElements.value.length > 0 && draggedIndex !== -1) {
        void dismissActiveOrb(direction, draggedIndex);
      } else {
        if (draggedIndex !== -1) {
            void bounceBackOrb(draggedIndex);
        }
      }
    } else if (touchId1 !== null && touchOrigin1 !== null) {
       // Tap check
      const touch = getTouchById(event.changedTouches, touchId1);
      if (touch !== null) {
        const deltaX = touch.clientX - touchOrigin1.clientX;
        const deltaY = touch.clientY - touchOrigin1.clientY;
        const dist = Math.hypot(deltaX, deltaY);
        // Truly empty: no gallery items AND no items with status left/right
        const isTrulyEmpty = galleryItems.value.length === 0 && !galleryItems.value.some(item => item.status === 'left' || item.status === 'right');
        if (dist < CLICK_MOVE_THRESHOLD_PX && orbs.value.length === 0 && isTrulyEmpty) {
          emit('upload-click');
        }
      }
    }
    
    isVerticalDrag.value = false;
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
  // Snap after touch interaction end
  // (handled by physics)
  isUserPressing.value = false;
  touchId1 = null;
  touchPrev1 = null;
  touchOrigin1 = null;
  
  // Engage Target Locking
  resolveScrollTarget();
  
  // After resolveScrollTarget: if all orbs dismissed, scroll to gallery (overrides snap-to-0)
  if (orbs.value.length === 0 && galleryItems.value.some(item => item.status === 'left' || item.status === 'right')) {
    scrollAnchorVel.value = 0;
    scrollTarget.value = 1;
    pendingScrollToGallery.value = false;
  }
}

function touchCancelCallback() {
  isUserPressing.value = false;
  isVerticalDrag.value = false;
  isUserPressing.value = false;
  touchId1 = null;
  touchPrev1 = null;
  touchOrigin1 = null;
  pinchPrevDist = null;
}

const scrollToOrb = (index: number) => {
  scrollAnchor.value = index;
  scrollAnchorVel.value = 0;
};

const scrollToGalleryItem = (item: GalleryItem) => {
  if (item.status !== 'active') return;
  const index = orbs.value.findIndex(o => o.galleryId === item.id);
  if (index !== -1) {
    scrollToOrb(index);
  }
};

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
  
  // Initial layout calculation
  updateLayout();
  window.addEventListener('resize', updateLayout);

  // Watch for gallery view state
  watch(() => Math.round(scrollAnchor.value), (current) => {
    const isOnGallery = current >= orbs.value.length + 1;
    emit('gallery-view-change', isOnGallery);
  });
  
  // Watch for active orbs count
  watch(() => orbs.value.length, (count) => {
    emit('active-orbs-change', count);
    
    // When all orbs are dismissed, smoothly scroll to gallery (results)
    if (count === 0) {
      const hasDismissedOrbs = galleryItems.value.some(item => item.status === 'left' || item.status === 'right');
      if (hasDismissedOrbs) {
        pendingScrollToGallery.value = false;
        // Smooth spring animation to gallery position
        scrollAnchorVel.value = 0;
        scrollTarget.value = 1; // gallery sits at orbs.length + 1 = 0 + 1 = 1
      }
    }
  }, { immediate: true });


  interactionElement.addEventListener('mousedown', (mouseEvent) => {
    // Left mouse button only
    if (mouseEvent.button !== 0) {
      return;
    }
    // Prevent default dragging/selection behavior so drag only controls rotation
    mouseEvent.preventDefault();
    const pos = { x: mouseEvent.clientX, y: mouseEvent.clientY };
    
    // Initialize new drag state
    isVerticalDrag.value = false;
    hasDragMoved.value = false;
    dragStartY.value = mouseEvent.clientY;
    dragStartScrollOffset.value = scrollAnchor.value;

    const targetOrbIndex = getOrbIndexAtPoint(pos);
    const shouldOrbDrag =
      uploadedImageElements.value.length > 0 &&
      !isOrbDismissing.value &&
      targetOrbIndex !== -1;
      
    if (shouldOrbDrag) {
      activeOrbDragIndex.value = targetOrbIndex;
      orbDragMouseActive = true;
      orbDragMouseStart = pos;
      orbDragLastX = pos.x;
      isOrbDragging.value = true;
      mouseStartPosition = null;
    } else {
      activeOrbDragIndex.value = null;
      orbDragMouseActive = false;
      orbDragMouseStart = null;
      isOrbDragging.value = false;
      mouseStartPosition = pos;
    }
    isUserPressing.value = true;
    scrollAnchorVel.value = 0; // Stop existing scroll momentum
  });
  
  let mousePrevTime = performance.now();
  document.addEventListener('mousemove', (mouseEvent) => {
    const now = performance.now();
    
    // If not pressing, just return (or if processing other drags)
    if (!isUserPressing.value) return; 
    
    const dt = Math.max(1, now - dragLastTime.value);
    const dy = mouseEvent.clientY - dragLastY.value;

    // 1:1 Direct Vertical Drag
    const totalDragY = mouseEvent.clientY - dragStartY.value;
    const progressDelta = -totalDragY / ORB_SPACING.value;
    scrollAnchor.value = dragStartScrollOffset.value + progressDelta;
    
    // Update velocity for momentum
    scrollAnchorVel.value = (-dy / ORB_SPACING.value) / (dt / 1000); // units per sec
    
    // 2. Horizontal Drag (Rotation / Dismiss) - Only if dragging a specific orb
    if (orbDragMouseActive && orbDragMouseStart !== null && activeOrbDragIndex.value !== null) {
      const index = activeOrbDragIndex.value;
      const orb = orbs.value[index];
      if (orb) {
          // Rotate kaleidoscope incrementally based on horizontal drag delta
          const prevX = orbDragLastX ?? mouseEvent.clientX;
          const dx = mouseEvent.clientX - prevX;
          const dtMove = Math.max(1, now - mousePrevTime);
          
          orb.rotation += dx / 300;
          orb.rotationVel = clampRotationVelocity((dx / dtMove) * 0.02);
          
          orbDragLastX = mouseEvent.clientX;
          setOrbDragVisuals(mouseEvent.clientX - orbDragMouseStart.x);
          mousePrevTime = now;
      }
    }
    
    hasDragMoved.value = true;
    
    mousePrevTime = now;
    // Keep drag tracker updated for consistency
    dragLastY.value = mouseEvent.clientY;
    dragLastTime.value = now;
  });
  
  document.addEventListener('mouseup', (mouseEvent: MouseEvent) => {
    
    // Unified Mouse Up Logic
    
    // Check dismiss / bounce for active orb
    if (orbDragMouseActive && orbDragMouseStart !== null) {
      const deltaX = mouseEvent.clientX - orbDragMouseStart.x;
      const threshold = getCenterOrbDismissThresholdPx();
      const direction: 1 | -1 = deltaX >= 0 ? 1 : -1;
      
      const draggedIndex = activeOrbDragIndex.value ?? -1;
      activeOrbDragIndex.value = null;

      orbDragMouseActive = false;
      orbDragMouseStart = null;
      isOrbDragging.value = false;
      
      if (Math.abs(deltaX) >= threshold && uploadedImageElements.value.length > 0 && draggedIndex !== -1) {
        void dismissActiveOrb(direction, draggedIndex);
      } else {
        // Reset drag visuals with bounce back
        if (draggedIndex !== -1) {
           void bounceBackOrb(draggedIndex);
        }
      }
    } else if (mouseStartPosition !== null) {
      // Background click check
      const deltaX = mouseEvent.clientX - mouseStartPosition.x;
      const deltaY = mouseEvent.clientY - mouseStartPosition.y;
      const dist = Math.hypot(deltaX, deltaY);
      
      // Truly empty: no gallery items AND no items with status left/right
      const isTrulyEmpty = galleryItems.value.length === 0 && !galleryItems.value.some(item => item.status === 'left' || item.status === 'right');
      if (dist < CLICK_MOVE_THRESHOLD_PX && orbs.value.length === 0 && isTrulyEmpty) {
        emit('upload-click');
      }
    }
    
    // Cleanup shared state
    isVerticalDrag.value = false;
    isUserPressing.value = false;
    mouseStartPosition = null;
    
    // Engage Target Locking
    resolveScrollTarget();
    
    // After resolveScrollTarget: if all orbs dismissed, scroll to gallery (overrides snap-to-0)
    if (orbs.value.length === 0 && galleryItems.value.some(item => item.status === 'left' || item.status === 'right')) {
      scrollAnchorVel.value = 0;
      scrollTarget.value = 1;
      pendingScrollToGallery.value = false;
    }
  });

  // Wheel Timeout for locking
  let wheelTimeout: any = null;

  document.addEventListener('wheel', (wheelEvent) => {
    wheelEvent.preventDefault();
    if (Math.abs(wheelEvent.deltaY) < 4) {
      return;
    }
    const clampedDelta = Math.max(-120, Math.min(120, wheelEvent.deltaY));
    const isTrackpad = wheelEvent.deltaMode === 0;
    // Compute impulse from raw delta (single mapping, nonlinear)
    const progressDelta = computeImpulseFromDelta(clampedDelta, isTrackpad);
    
    // Keep target active, update velocity
    // No cap, just raw accumulation
    scrollAnchorVel.value += progressDelta * debugFlickMultiplier.value;
    
    // Update target immediately
    resolveScrollTarget();
    
    // Schedule lock (as backup)
    if (wheelTimeout) clearTimeout(wheelTimeout);
    wheelTimeout = setTimeout(() => {
        resolveScrollTarget();
    }, WHEEL_LOCK_TIMEOUT_MS);
    
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
    
    // Apply global scroll impulse to all orbs
    orbs.value.forEach(orb => {
        orb.rotationVel = clampRotationVelocity(orb.rotationVel + velocityImpulse);
    });
    
    lastWindowScrollY = window.scrollY;
    lastWindowScrollTime = now;
  };
  window.addEventListener('scroll', onWindowScroll, { passive: true });
  onUnmounted(() => {
    window.removeEventListener('scroll', onWindowScroll);
    window.removeEventListener('resize', updateLayout);
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
      // removeUploadedImageAtIndex already handled orb removal and set scrollTarget
      // for smooth gallery transition — let the spring physics do its work.
      return;
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
    orbs.value = [];
    activeImageIndex.value = 0;
    // Dismissed orbs: any items with status 'left' or 'right'
    const hasDismissedOrbs = galleryItems.value.some(item => item.status === 'left' || item.status === 'right');
    // Truly empty: no gallery items AND no items with status left/right
    const isTrulyEmpty = galleryItems.value.length === 0 && !hasDismissedOrbs;

    // When there are dismissed orbs (gallery), smooth-scroll to gallery screen; otherwise reset
    if (hasDismissedOrbs) {
      // Use scrollTarget so the spring physics animate smoothly to the gallery position
      scrollAnchorVel.value = 0;
      scrollTarget.value = 1; // gallery sits at orbs.length + 1 = 0 + 1 = 1
      // Do NOT restart camera when viewing gallery
    } else {
      scrollAnchor.value = 0;
      // Webcam is not used; no camera restart.
    }
  }
}, { immediate: false });

</script>

<template>
  <!-- Full-viewport wrapper so #app has height and toolbar stays at bottom -->
  <div class="w-[100dvw] h-[100dvh] relative overflow-hidden">
    <div
      ref="interaction-layer"
      class="absolute inset-0 z-[200] touch-none select-none"
    />

    <!-- Dynamic Orbs List -->
    <div 
       v-for="orb in orbs" 
       :key="orb.id"
       class="absolute left-1/2 overflow-hidden rounded-full bg-neutral-800 w-[64vmin] h-[64vmin] min-w-[160px] min-h-[160px]"
       :style="getOrbStyle(orb)"
    >
       <canvas
          :ref="(el) => setCanvasRef(el, orb.id)"
          class="block w-full h-full object-cover"
       />
    </div>

    <!-- Grid Gallery -->
    <div
       v-if="galleryItems.length > 0"
       class="absolute left-1/2 w-[90vw] max-w-md grid grid-cols-3 gap-3 p-4 transition-all duration-500 ease-out"
       :style="{
          top: '50%',
          left: '50%',
          transform: `translate(-50%, -50%) translateY(${(orbs.length + 1 - scrollAnchor) * ORB_SPACING}px)`,
          opacity: Math.abs(orbs.length + 1 - scrollAnchor) < 0.5 ? 1 : 0,
          zIndex: 100,
          pointerEvents: scrollAnchor > orbs.length - 0.5 ? 'auto' : 'none'
       }"
    >
       <div 
         v-for="item in galleryItems" 
         :key="item.id"
         class="relative aspect-square rounded-full overflow-hidden transition-transform bg-neutral-800"
         :class="[
            item.status === 'active' ? 'cursor-pointer hover:scale-105 active:scale-95 opacity-70' : ''
         ]"
         @click.stop="scrollToGalleryItem(item)"
       >
          <!-- All items show kaleidoscope thumbnail -->
          <img
             :src="item.thumbnailSrc || item.src"
             class="block w-full h-full object-cover"
          />
          
          <!-- X Overlay for left swipes -->
          <div 
            v-if="item.status === 'left'" 
            class="absolute inset-0 flex items-center justify-center"
          >
             <svg xmlns="http://www.w3.org/2000/svg" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="white" stroke-width="3" stroke-linecap="round" stroke-linejoin="round" class="drop-shadow-lg"><path d="M18 6 6 18"/><path d="m6 6 12 12"/></svg>
          </div>
          
          <!-- Checkmark Overlay for right swipes -->
          <div 
            v-if="item.status === 'right'" 
            class="absolute inset-0 flex items-center justify-center"
          >
             <svg xmlns="http://www.w3.org/2000/svg" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="white" stroke-width="3" stroke-linecap="round" stroke-linejoin="round" class="drop-shadow-lg"><path d="M20 6 9 17l-5-5"/></svg>
          </div>
       </div>
    </div>

    <!-- Hidden WebGL canvas used as source -->
    <canvas
      id="maincanvas"
      ref="canvas"
      class="absolute -left-[9999px] -top-[9999px] opacity-0 pointer-events-none"
    />
  </div>


</template>

<style scoped>
</style>
