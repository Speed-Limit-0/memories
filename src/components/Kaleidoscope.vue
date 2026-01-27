<script setup lang="ts">
import {ref, onMounted, useTemplateRef, watch} from 'vue';
import { ScopeShape } from '../scopeShape.ts';

const props = defineProps<{
  scopeShape: ScopeShape,
  scopeAutoRotationVelocity: number
  saveNextFrame: boolean
  uploadedImage?: string | null
}>();

const emit = defineEmits(['save-frame']);

const facingMode = ref('unknown');
const cameraZoom = ref(1);
const scopeRotation = ref(0.0);
const scopeSize = ref(0.5);
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
const uploadedImageElement = ref(null as HTMLImageElement | null);
let cameraStream: MediaStream | null = null;
let textureNeedsUpdate = ref(true);

const loadUploadedImage = async (imageSrc: string | null) => {
  if (imageSrc) {
    const img = new Image();
    img.crossOrigin = 'anonymous';
    img.src = imageSrc;
    await new Promise((resolve, reject) => {
      img.onload = resolve;
      img.onerror = reject;
    });
    uploadedImageElement.value = img;
    facingMode.value = 'user'; // Default for uploaded images
    textureNeedsUpdate.value = true; // Mark texture for update
    
    // Stop camera stream if image is uploaded
    if (cameraStream) {
      cameraStream.getTracks().forEach(track => track.stop());
      cameraStream = null;
    }
  } else {
    uploadedImageElement.value = null;
    textureNeedsUpdate.value = true; // Mark texture for update when switching back to camera
  }
};

async function main() {
  // Capture webcam input using invisible `video` element
  // Adapted from p5js.org/examples/3d-shader-using-webcam.html
  const camera = document.getElementById('camera') as HTMLVideoElement;

  // If an image is uploaded, create an image element for it
  if (props.uploadedImage) {
    await loadUploadedImage(props.uploadedImage);
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

  // Canvas with WebGL context
  const canvas = document.getElementById('maincanvas') as HTMLCanvasElement;
  const canvasSize = Math.max(1024, window.innerWidth, window.innerHeight) * window.devicePixelRatio;
  const gl = canvas.getContext('webgl')!;
  canvas.width = canvas.height = canvasSize;
  gl.viewport(0, 0, canvas.width, canvas.height);

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
      uniform vec2 dataDimensions;
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

      vec2 isosceles(vec2 u, float kLength, float kRot, vec2 offset) {
        // Center the triangle in a circle
        u -= vec2(0.5, 0.5);

        u = rotate2d(u,kRot);
        u /= kLength;
        u *= sqrt(2.0) / 2.0;

        u += vec2(0.5, 0.5);
        u -= 0.25;

        u += offset;

        vec2 k = vec2(0.0, 0.0);

        vec2 squareCentroid = vec2(round(u.x), round(u.y));
        // For debugging
        // k = vec2(distance(squareCentroid, u));

        float distance = distance(squareCentroid, u) * 2.0;
        float deg180 = 3.1415926536;
        float deg45 = deg180 / 4.0;
        // Multiply by 0.99999, because atan2 fails on some corner cases :(
        float theta = atan(u.x- squareCentroid.x, u.y- squareCentroid.y) * 0.999999;
        if (mod(theta, deg45 * 2.0) > deg45) {
          theta = deg45 - mod(theta, deg45);
        } else {
          theta = mod(theta, deg45);
        }

        k.x = cos(theta) * distance;
        k.y = sin(theta) * distance;

        return k;
      }

      vec2 scalene(vec2 u, float kLength, float kRot, vec2 offset) {
        u -= 0.5;

        u = rotate2d(u,kRot + radians(60.0));
        u /= kLength;

        u += 0.5;

        u.x -= sin(radians(90.0)) * 0.50;
        u.y -= cos(radians(90.0)) * 0.50;

        u += offset;

        vec2 k = vec2(0.0, 0.0);

        vec2 hexIndex = square_float_to_axial_hex_grid(u, true);
        vec2 hexCentroid = hexToCentroid(hexIndex, true);
        // For debugging
        // k = vec2(distance(hexToCentroid(hexIndex, true), u));
        // k = hexIndex / 5.0;

        float distance = distance(hexCentroid, u) * 2.0 / sqrt(3.0);
        float deg180 = 3.1415926536;
        float deg30 = deg180 / 6.0;
        float theta = atan(u.x- hexCentroid.x, u.y- hexCentroid.y) + deg180;
        if (mod(theta, deg30 * 2.0) > deg30) {
          theta = mod(theta, deg30);
        } else {
          theta = deg30 - mod(theta, deg30);
        }

        k.x = cos(theta) * distance;
        k.y = sin(theta) * distance;

        // We want the center of the triangle to be the center of the image.
        k *= sqrt(3.0) / 2.0;
        k.x += (((1.0 - sqrt(3.0) / 2.0)) / 2.0);
        k.y += (0.25);

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
          } else if (scopeShape == ${ScopeShape.Isosceles}) {
            k = isosceles(k, scopeSize, scopeRotation, d);
          } else {
            scopeDiameterRatio = 1.0;
            k = scalene(k, scopeSize, scopeRotation, d);
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
          
          // Refraction/distortion at edges of each reflection - warp outward from segment center
          // Distortion should ONLY apply at edges, not at center
          // Create a distortion factor that's zero at center and increases toward edges
          float distortionFactor = 1.0 - smoothstep(0.0, 0.4, distToNearestEdge); // Zero at center, 1 at edges
          distortionFactor = pow(distortionFactor, 0.8); // Make it more edge-focused
          
          // Use a smoother distortion direction to avoid visible lines along diagonals
          // Rotate the direction slightly to break up diagonal patterns
          vec2 distortionDir = normalize(toSegmentCenter + vec2(0.001)); // Avoid division by zero
          distortionDir = rotate2d(distortionDir, 0.1); // Slight rotation to smooth out diagonal lines
          
          // Reduce distortion for square-based shapes (Square and Isosceles) to avoid aggressive artifacts
          float distortionMultiplier = 2.5;
          if (scopeShape == ${ScopeShape.Square} || scopeShape == ${ScopeShape.Isosceles}) {
            distortionMultiplier = 1.2; // Much less aggressive for square-based patterns
          }
          float distortionStrength = distortionFactor * distortionMultiplier;
          // Scale by distance from center to make it stronger further from center
          float distortionScale = smoothstep(0.3, maxSegmentDist, distFromSegmentCenter); // Start later to avoid center artifacts
          // Apply smoother, more gradual distortion
          vec2 distortedK = k + distortionDir * distortionStrength * distortionScale * smoothstep(0.0, 1.0, distFromSegmentCenter / maxSegmentDist);
          
          // Now map the k value to coordinates on the image
          // 0,0 will be the centre of the image
          // 1,1 will be the top right of the image (not the bottom left– It's easier to orientate if things are up-right)
          float dataMinDimension = min(dataDimensions.x, dataDimensions.y) / scopeDiameterRatio;
          float dataWindowSize = dataMinDimension * dataScopePercentage;
          vec2 i = vec2(0.0,0.0);
          // x-axis is flipped only when the camera is pointed to the user
          i.x = (-dataWindowSize / 2.0 + distortedK.x * dataWindowSize) * (dataIsFacingUser == 1 ? -1.0 : 1.0);
          // y-axis is flipped because of openGL coordinate space
          i.y = - (-dataWindowSize / 2.0 + distortedK.y * dataWindowSize);
          i = rotate2d(i, scopeRotation * (dataIsFacingUser == 1 ? -1.0 : 1.0));
          i /= dataZoom;
          
          // Chromatic aberration - sample RGB channels at slightly offset positions
          // Offset direction is radial from segment center
          float aberrationStrength = edgeFactor * 0.02; // Reduced aberration strength (in texture coordinate space)
          vec2 aberrationDir = normalize(rotate2d(toSegmentCenter, scopeRotation) + vec2(0.001)); // Avoid division by zero
          
          vec2 iR = i + aberrationDir * aberrationStrength * dataWindowSize;
          vec2 iG = i;
          vec2 iB = i - aberrationDir * aberrationStrength * dataWindowSize;
          
          iR.x += dataDimensions.x / 2.0;
          iR.y += dataDimensions.y / 2.0;
          iG.x += dataDimensions.x / 2.0;
          iG.y += dataDimensions.y / 2.0;
          iB.x += dataDimensions.x / 2.0;
          iB.y += dataDimensions.y / 2.0;
          
          vec2 texCoordR = clamp(vec2(iR.x, iR.y) / vec2(dataDimensions.x, dataDimensions.y), 0.0, 1.0);
          vec2 texCoordG = clamp(vec2(iG.x, iG.y) / vec2(dataDimensions.x, dataDimensions.y), 0.0, 1.0);
          vec2 texCoordB = clamp(vec2(iB.x, iB.y) / vec2(dataDimensions.x, dataDimensions.y), 0.0, 1.0);
          
          float r = texture2D(data, texCoordR).r;
          float g = texture2D(data, texCoordG).g;
          float b = texture2D(data, texCoordB).b;
          float a = texture2D(data, texCoordG).a;
          
          // Vignetting - progressively darken edges of each reflection segment
          // Radial vignette: darkens in a circular pattern from center outward
          float vignetteFactor = smoothstep(0.0, maxSegmentDist, distFromSegmentCenter); // Radial distance from center
          float vignette = 1.0 - vignetteFactor * 0.6; // Progressive darkening (60% darker at edges)
          vignette = max(vignette, 0.05); // Keep minimum brightness
          
          // Progressive dimming based on tile distance from center
          // This makes each reflection copy dimmer as it gets further from the center tile
          // tileDistance represents how many tiles away from center we are
          float dimmingFactor = smoothstep(0.0, 2.0, tileDistance); // Dim over first 2 tiles from center (starts earlier)
          dimmingFactor = pow(dimmingFactor, 0.4); // Smoother, earlier falloff
          
          // Dim reflections based on tile distance - each copy gets darker
          float dimming = 1.0 - dimmingFactor * 0.8; // Dim reflections (80% dimmer for distant tiles)
          dimming = max(dimming, 0.2); // Keep minimum brightness (20% for very distant tiles)
          
          // Progressive blur based on tile distance (same as dimming)
          // Blur strength increases as reflections get further from center
          float blurFactor = smoothstep(0.0, 2.0, tileDistance); // Blur over first 2 tiles from center
          blurFactor = pow(blurFactor, 0.4); // Same falloff as dimming
          float blurStrength = blurFactor * 0.025; // Maximum blur strength (adjustable)
          
          // Sample texture at multiple offset positions for blur effect
          vec2 texCoordCenter = vec2(iG.x, iG.y) / vec2(dataDimensions.x, dataDimensions.y);
          vec2 blurOffset = vec2(blurStrength, 0.0);
          
          // Simple box blur - sample 9 points in a 3x3 grid
          vec3 blurredColor = vec3(0.0);
          float sampleCount = 0.0;
          for (float x = -1.0; x <= 1.0; x += 1.0) {
            for (float y = -1.0; y <= 1.0; y += 1.0) {
              vec2 offset = vec2(x, y) * blurStrength;
              vec2 sampleCoord = clamp(texCoordCenter + offset, 0.0, 1.0);
              vec4 sample = texture2D(data, sampleCoord);
              blurredColor += sample.rgb;
              sampleCount += 1.0;
            }
          }
          blurredColor /= sampleCount;
          
          // Blend between sharp and blurred based on blur strength
          vec3 finalColor = mix(vec3(r, g, b), blurredColor, blurFactor);
          
          gl_FragColor = vec4(finalColor * vignette * dimming, a);

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

  // Texture to contain the video data
  const texture = gl.createTexture();
  gl.bindTexture(gl.TEXTURE_2D, texture);
  gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, true);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);

  // Bind texture to the "data" argument to the fragment shader
  gl.uniform1i(gl.getUniformLocation(program,'data'),0);
  gl.activeTexture(gl.TEXTURE0);
  gl.bindTexture(gl.TEXTURE_2D,texture);

  // Bind camera dimensions to the fragment shader
  const dataDimensionsBind = gl.getUniformLocation(program, 'dataDimensions');
  const dataIsFacingUserBind = gl.getUniformLocation(program, 'dataIsFacingUser');
  const dataZoomBind = gl.getUniformLocation(program, 'dataZoom');
  const canvasDimensionsBind = gl.getUniformLocation(program, 'canvasDimensions');
  const scopeShapeBind = gl.getUniformLocation(program, 'scopeShape');
  const scopeRotationBind = gl.getUniformLocation(program, 'scopeRotation');
  const scopeSizeBind = gl.getUniformLocation(program, 'scopeSize');
  const scopeOffsetBind = gl.getUniformLocation(program, 'scopeOffset');

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
  function animate(){
    // Handle keyboard state
    if (keyPressedA.value && keyPressedD.value) {
      // Do nothing
    } else if (keyPressedA.value) {
      scopeRotationVel.value -= fastNormalSlow(0.001, 0.0002, 0.00005);
    } else if (keyPressedD.value) {
      scopeRotationVel.value += fastNormalSlow(0.001, 0.0002, 0.00005);
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
    } else if (props.scopeShape === ScopeShape.Isosceles) {
      scopeRotationOffset = -Math.PI / 2;
    } else if (props.scopeShape === ScopeShape.Scalene) {
      scopeRotationOffset = -Math.PI / 2;
    }

    // A hack to fix an issue with the scope offset calculations for a specific shape...
    if (props.scopeShape === ScopeShape.Scalene) {
      scopeOffset.value[0] += Math.sin(-scopeRotation.value - scopeRotationOffset - Math.PI / 3) * scopeOffsetVel.value[0] - Math.cos(-scopeRotation.value - scopeRotationOffset - Math.PI / 3) * scopeOffsetVel.value[1];
      scopeOffset.value[1] += Math.cos(-scopeRotation.value - scopeRotationOffset - Math.PI / 3) * scopeOffsetVel.value[0] + Math.sin(-scopeRotation.value - scopeRotationOffset - Math.PI / 3) * scopeOffsetVel.value[1];
    } else {
      scopeOffset.value[0] += Math.sin(-scopeRotation.value - scopeRotationOffset) * scopeOffsetVel.value[0] - Math.cos(-scopeRotation.value - scopeRotationOffset) * scopeOffsetVel.value[1];
      scopeOffset.value[1] += Math.cos(-scopeRotation.value - scopeRotationOffset) * scopeOffsetVel.value[0] + Math.sin(-scopeRotation.value - scopeRotationOffset) * scopeOffsetVel.value[1];
    }

    if (props.scopeAutoRotationVelocity !== 0) {
      scopeRotationVel.value = props.scopeAutoRotationVelocity / 25;
      scopeRotation.value += scopeRotationVel.value;
    }
    if (touchOrigin1 === null && mousePrevPosition === null) {
      scopeRotation.value += scopeRotationVel.value;
      scopeRotationVel.value *= 0.99;
      scopeSizeVel.value *= 0.95;
      scopeSize.value *= 1 + Math.min(scopeSizeVel.value, 0.99);
      if (scopeSize.value > 2.0) {
        scopeSize.value -= (scopeSize.value - 2) / 10;
      }
      if (scopeSize.value < Math.pow(0.5, 7)) {
        scopeSize.value += (Math.pow(0.5, 7) - scopeSize.value) / 10;
      }
    }
    scopeOffsetVel.value[0] *= 0.95;
    scopeOffsetVel.value[1] *= 0.95;

    // Use uploaded image if available, otherwise use camera
    const imageSource = uploadedImageElement.value || camera;
    let imageWidth: number;
    let imageHeight: number;
    
    if (uploadedImageElement.value) {
      imageWidth = uploadedImageElement.value.width;
      imageHeight = uploadedImageElement.value.height;
    } else {
      imageWidth = camera.videoWidth || 1;
      imageHeight = camera.videoHeight || 1;
    }

    // Only render if we have valid dimensions
    if (imageWidth > 0 && imageHeight > 0) {
      // Only update texture if:
      // 1. Using video (which changes every frame), OR
      // 2. Texture needs update (new image uploaded or switched back to camera)
      const isVideo = !uploadedImageElement.value;
      if (isVideo || textureNeedsUpdate.value) {
        gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, imageSource);
        textureNeedsUpdate.value = false; // Reset flag after update
      }
      gl.uniform2f(dataDimensionsBind, imageWidth, imageHeight);
    gl.uniform1i(dataIsFacingUserBind, facingMode.value === 'user' ? 1 : 0);
    gl.uniform1f(dataZoomBind, cameraZoom.value);
    gl.uniform1i(scopeShapeBind, props.scopeShape);
    gl.uniform1f(scopeRotationBind, scopeRotation.value + scopeRotationOffset);
    gl.uniform1f(scopeSizeBind, scopeSize.value);
    gl.uniform2f(scopeOffsetBind, scopeOffset.value[0], scopeOffset.value[1]);
      gl.uniform2f(canvasDimensionsBind, canvasSize, canvasSize);
      gl.drawArrays(gl.TRIANGLES, 0, 6);
    }

    if (props.saveNextFrame) {
      emit(
        'save-frame',
        canvas.toDataURL('image/jpeg', 0.8)
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
let touchId1: null|number = null;
let touchOrigin1: null|Point = null;
let touchPrevTime = new Date().getTime();
let touchPrev1: null|Point = null;

function getTouchById(touches: TouchList, id: number): Touch | null {
  for (let i = 0; i < touches.length; i += 1) {
    if (touches[i].identifier === id) {
      return touches[i];
    }
  }
  return null;
}

function touchStartCallback(event: TouchEvent) {
  event.preventDefault();
  if (touchId1 !== null) {
    return;
  }
  const touch = event.changedTouches[0];
  touchId1 = touch.identifier;
  touchPrev1 = touch;
  touchOrigin1 = touch;
  touchPrevTime = new Date().getTime();
  isUserPressing.value = true;
  scopeRotationVel.value = 0;
  scopeSizeVel.value = 0;
}

function touchMoveCallback(event: TouchEvent) {
  if (touchId1 === null || touchPrev1 === null || touchOrigin1 === null) {
    return;
  }
  const touch = getTouchById(event.changedTouches, touchId1);
  if (touch === null) {
    return;
  }
  if (new Date().getTime() - touchPrevTime < 0.001) {
    return;
  }
  const deltaTime = Math.max(new Date().getTime() - touchPrevTime, 0.001);
  const deltaX = (touch.clientX - touchPrev1.clientX) / deltaTime;
  const deltaY = (touch.clientY - touchPrev1.clientY) / deltaTime;

  scopeRotation.value += deltaX / 10;
  if (Math.abs(touch.clientX - touchPrev1.clientX) > 1) {
    scopeRotationVel.value = deltaX / 10;
  } else {
    scopeRotationVel.value = 0;
  }

  scopeSize.value *= 1.0 + deltaY / 10;
  if (Math.abs(touch.clientY - touchPrev1.clientY) > 1) {
    scopeSizeVel.value = deltaY / 10;
  } else {
    scopeSizeVel.value = 0;
  }

  touchPrevTime = new Date().getTime();
  touchPrev1 = touch;
}

function touchEndCallback(event: TouchEvent) {
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
}

onMounted(() => {
  main();

  const canvasElement = canvas.value as HTMLCanvasElement;

  canvasElement.addEventListener('mousedown', (mouseEvent) => {
    // Left mouse button only
    if (mouseEvent.button !== 0) {
      return;
    }
    mousePrevPosition = {
      x: mouseEvent.clientX,
      y: mouseEvent.clientY,
    };
    isUserPressing.value = true;
    scopeRotationVel.value = 0;
    scopeSizeVel.value = 0;
  });
  document.addEventListener('mousemove', (mouseEvent) => {
    if (mousePrevPosition === null) {
      return;
    }
    const deltaX = (mouseEvent.clientX - mousePrevPosition.x) / 10;
    const deltaY = (mouseEvent.clientY - mousePrevPosition.y) / 10;

    scopeRotation.value += deltaX / 20;
    if (Math.abs(mouseEvent.clientX - mousePrevPosition.x) > 1) {
      scopeRotationVel.value = deltaX / 20;
    } else {
      scopeRotationVel.value = 0;
    }

    scopeSize.value *= 1.0 + deltaY / 50;
    if (Math.abs(mouseEvent.clientY - mousePrevPosition.y) > 1) {
      scopeSizeVel.value = deltaY / 50;
    } else {
      scopeSizeVel.value = 0;
    }

    mousePrevPosition = {
      x: mouseEvent.clientX,
      y: mouseEvent.clientY,
    };
  });
  document.addEventListener('mouseup', () => {
    if (mousePrevPosition === null) {
      return;
    }

    isUserPressing.value = false;
    mousePrevPosition = null;
  });

  document.addEventListener('wheel', (wheelEvent) => {
    wheelEvent.preventDefault();
    scopeRotation.value += wheelEvent.deltaX / 500;
    scopeSize.value *= 1.0 - wheelEvent.deltaY / 500;
  });

  canvasElement.addEventListener('touchstart', touchStartCallback);
  canvasElement.addEventListener('touchmove', touchMoveCallback);
  canvasElement.addEventListener('touchend', touchEndCallback);
  canvasElement.addEventListener('touchcancel', touchCancelCallback);

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

// Watch for changes to uploaded image
watch(() => props.uploadedImage, async (newImage) => {
  if (newImage) {
    await loadUploadedImage(newImage);
  } else {
    uploadedImageElement.value = null;
    // Restart camera if no image is uploaded
    const camera = document.getElementById('camera') as HTMLVideoElement | null;
    if (camera && !cameraStream) {
      try {
        cameraStream = await navigator.mediaDevices.getUserMedia({video: { facingMode: { exact: 'environment'} }, audio: false});
        facingMode.value = cameraStream.getVideoTracks()[0]?.getSettings().facingMode ?? 'user';
      } catch (e) {
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
});

</script>

<template>
  <canvas
    id="maincanvas"
    ref="canvas"
    class="bg-black"
    style="width:100dvw;height:100dvh;object-fit:cover"
  />
  <video
    id="camera"
    visible="False"
    style="width: 512px; height: 512px; display:none;"
    controls="true"
    playsinline
    crossorigin="anonymous"
  />
</template>

<style scoped>
</style>
