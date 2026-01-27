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
          
          // Circular mask - calculate distance from center of screen
          vec2 screenCenter = vec2(0.5, 0.5);
          vec2 screenPos = vec2(fragCoord.x, 1.0 - fragCoord.y); // Account for flipped y-coordinate
          float distFromCenter = distance(screenPos, screenCenter);
          float circleRadius = 0.25; // Radius of the circle (smaller than half the screen)
          // Hard edge cutoff - no smoothstep to eliminate halo completely
          float circleMask = step(distFromCenter, circleRadius); // Hard edge, no transparency gradient
          
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
          vec2 circleDistortionDir = normalize(screenPos - screenCenter + vec2(0.001)); // Avoid division by zero
          circleDistortionDir = rotate2d(circleDistortionDir, 0.1); // Slight rotation to smooth out patterns
          
          // Apply stronger distortion to texture coordinates based on circular container edge
          // Distortion pushes outward from center, creating a "lens" or "fisheye" effect at edges
          float circleDistortionStrength = circleDistortionFactor * 5.0; // Increased distortion strength
          vec2 circleDistortionOffset = circleDistortionDir * circleDistortionStrength * dataWindowSize * 0.03;
          
          // Chromatic aberration - only apply at circular container edges
          // Offset direction is radial from circle center (not segment center)
          // Only apply aberration when inside the circular container
          float aberrationStrength = circleDistortionFactor * circleMask * 0.02; // Reduced aberration strength (in texture coordinate space)
          vec2 aberrationDir = normalize(rotate2d(screenPos - screenCenter, scopeRotation) + vec2(0.001)); // Avoid division by zero
          
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
          
          // Get original sharp color with chromatic aberration and circular edge distortion
          float r = texture2D(data, texCoordR).r;
          float g = texture2D(data, texCoordG).g;
          float b = texture2D(data, texCoordB).b;
          float a = texture2D(data, texCoordG).a;
          
          // Subtle blur - only at edges, gentle in center
          float blurFactor = pow(circleDistortionFactor, 2.0); // Blur increases with distortion
          float blurStrength = blurFactor * 0.008; // Small blur strength
          
          // Simple Gaussian blur - sample in a small grid pattern
          vec2 texCoordCenter = texCoordG; // Use green channel as center
          vec3 blurredColor = vec3(0.0);
          float totalWeight = 0.0;
          
          // Sample in a 3x3 grid for subtle blur
          for (int x = -1; x <= 1; x++) {
            for (int y = -1; y <= 1; y++) {
              vec2 offset = vec2(float(x), float(y)) * blurStrength;
              vec2 sampleCoord = clamp(texCoordCenter + offset, 0.0, 1.0);
              
              // Gaussian weight
              float dist = length(vec2(float(x), float(y)));
              float weight = exp(-(dist * dist) / 0.5);
              
              vec4 sample = texture2D(data, sampleCoord);
              blurredColor += sample.rgb * weight;
              totalWeight += weight;
            }
          }
          
          if (totalWeight > 0.0) {
            blurredColor /= totalWeight;
          } else {
            blurredColor = vec3(r, g, b);
          }
          
          // Blend between sharp and blurred - subtle blend
          vec3 finalColor = mix(vec3(r, g, b), blurredColor, blurFactor * 0.6);
          
          // Limit reflections to main and adjacent tiles only (no infinite reflections)
          // tileDistance < 0.5 = main reflection, 0.5 <= tileDistance < 1.5 = adjacent reflections
          float maxTileDistance = 1.5; // Maximum tile distance to show
          float tileMask = 1.0 - smoothstep(maxTileDistance - 0.1, maxTileDistance, tileDistance); // Smooth edge
          
          // Apply vignette (which now fades before mask edge) and mask
          float combinedMask = circleMask * tileMask;
          gl_FragColor = vec4(finalColor * circleVignette, a * combinedMask);

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
    }

    scopeOffset.value[0] += Math.sin(-scopeRotation.value - scopeRotationOffset) * scopeOffsetVel.value[0] - Math.cos(-scopeRotation.value - scopeRotationOffset) * scopeOffsetVel.value[1];
    scopeOffset.value[1] += Math.cos(-scopeRotation.value - scopeRotationOffset) * scopeOffsetVel.value[0] + Math.sin(-scopeRotation.value - scopeRotationOffset) * scopeOffsetVel.value[1];

    if (props.scopeAutoRotationVelocity !== 0) {
      scopeRotationVel.value = props.scopeAutoRotationVelocity / 25;
      scopeRotation.value += scopeRotationVel.value;
    }
    if (touchOrigin1 === null && mousePrevPosition === null) {
      scopeRotation.value += scopeRotationVel.value;
      scopeRotationVel.value *= 0.99;
      scopeSizeVel.value *= 0.95;
      scopeSize.value = Math.max(0.2, Math.min(0.6, scopeSize.value * (1 + Math.min(scopeSizeVel.value, 0.99))));
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

  scopeSize.value = Math.max(0.2, Math.min(0.6, scopeSize.value * (1.0 + deltaY / 10)));
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

    scopeSize.value = Math.max(0.2, Math.min(0.6, scopeSize.value * (1.0 + deltaY / 50)));
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
    scopeSize.value = Math.max(0.2, Math.min(0.6, scopeSize.value * (1.0 - wheelEvent.deltaY / 500)));
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
    style="width:100dvw;height:100dvh;object-fit:cover;background-color:#EAEAE8"
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
