<script setup lang="ts">
import { ref, shallowRef, onMounted, onUnmounted } from 'vue';
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
// Optional: If you want controls
// import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

// --- Vue Refs ---
// Ref for the canvas element
const canvasRef = ref<HTMLCanvasElement | null>(null);

// ShallowRefs for Three.js objects (better performance than ref for complex objects)
const scene = shallowRef<THREE.Scene | null>(null);
const camera = shallowRef<THREE.PerspectiveCamera | null>(null);
const renderer = shallowRef<THREE.WebGLRenderer | null>(null);
const innerDisplay = shallowRef<THREE.Mesh | null>(null);
const outerDisplay = shallowRef<THREE.Mesh | null>(null);
const screen = shallowRef<THREE.Mesh | null>(null);
// Optional controls
// const controls = shallowRef<OrbitControls | null>(null);

// Ref to store the animation frame ID for cancellation
let animationFrameId: number | null = null;

// --- Three.js Initialization ---
const initThree = () => {
  if (!canvasRef.value) {
    console.error("Canvas element not found!");
    return;
  }

  // Scene
  scene.value = new THREE.Scene();
  scene.value.background = new THREE.Color(0xcccccc);

  // Camera
  camera.value = new THREE.PerspectiveCamera(
    75,
    window.innerWidth / window.innerHeight,
    0.1,
    1000
  );
  camera.value.position.set(2, 2, 5); // Adjust as needed

  // Renderer
  renderer.value = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    antialias: true,
  });
  renderer.value.setSize(window.innerWidth, window.innerHeight);
  renderer.value.setPixelRatio(window.devicePixelRatio);
  renderer.value.shadowMap.enabled = true;
  renderer.value.shadowMap.type = THREE.PCFSoftShadowMap;

  // Lighting
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
  scene.value.add(ambientLight);

  const directionalLight = new THREE.DirectionalLight(0xffffff, 1.0);
  directionalLight.position.set(5, 10, 7.5);
  directionalLight.castShadow = true;
  directionalLight.shadow.mapSize.width = 1024;
  directionalLight.shadow.mapSize.height = 1024;
  directionalLight.shadow.camera.near = 0.5;
  directionalLight.shadow.camera.far = 50;
  scene.value.add(directionalLight);

  // Optional Ground Plane
  const planeGeometry = new THREE.PlaneGeometry(20, 20);
  const planeMaterial = new THREE.MeshStandardMaterial({ color: 0xaaaaaa, side: THREE.DoubleSide });
  const plane = new THREE.Mesh(planeGeometry, planeMaterial);
  plane.rotation.x = -Math.PI / 2;
  plane.position.y = -1;
  plane.receiveShadow = true;
  scene.value.add(plane);

  // Optional Controls
  // controls.value = new OrbitControls(camera.value, renderer.value.domElement);
  // controls.value.enableDamping = true;

  // GLTF Loader
  const loader = new GLTFLoader();
  const path = "/models/laptopModel.glb"; // Ensure this path is correct relative to public folder

  loader.load(
    path,
    (gltf) => {
      console.log('Model loaded successfully!');
      const model = gltf.scene;

      model.traverse((node) => {
        if (node instanceof THREE.Mesh) {
          console.log("Traversing node:", node.name);
          node.castShadow = true;
          node.receiveShadow = true; // Optional

          // Use .value when assigning to shallowRef
          if (node.name === 'laptop_inner_display') {
            innerDisplay.value = node;
            console.log("Found inner display:", node.name);
          } else if (node.name === 'laptop_outer_display') {
            outerDisplay.value = node;
            console.log("Found outer display:", node.name);
          } else if (node.name === 'laptop_screen') {
            screen.value = node;
            console.log("Found screen:", node.name);
          }
        }
      });

      scene.value?.add(model); // Use optional chaining as scene.value might technically be null briefly
      console.log("Model added to scene. Starting animation loop.");
      // Start animation loop *after* model is loaded and added
      animate();
    },
    (xhr) => {
      console.log((xhr.loaded / xhr.total * 100) + '% loaded');
    },
    (error) => {
      console.error('An error happened loading the model:', error);
    }
  );

  // Add resize listener
  window.addEventListener('resize', handleResize);
};

// --- Resize Handler ---
const handleResize = () => {
  if (camera.value && renderer.value) {
    camera.value.aspect = window.innerWidth / window.innerHeight;
    camera.value.updateProjectionMatrix();
    renderer.value.setSize(window.innerWidth, window.innerHeight);
    renderer.value.setPixelRatio(window.devicePixelRatio);
  }
};

// --- Animation Loop ---
const animate = () => {
  // Ensure cleanup doesn't cancel an already stopped loop
  if (!renderer.value) return; 

  animationFrameId = requestAnimationFrame(animate);

  // Attempt Rotation (accessing .value of shallowRefs)
  if (innerDisplay.value) {
    const rotationSpeed = 0.005;
     // Basic clamping example - adjust as needed for your model's orientation
    if (innerDisplay.value.rotation.x > -Math.PI / 2 + 0.05 ) { // Add buffer to avoid getting stuck
        innerDisplay.value.rotation.x -= rotationSpeed; 
    }

    // Sync other parts if they exist
    if (outerDisplay.value) outerDisplay.value.rotation.x = innerDisplay.value.rotation.x;
    if (screen.value) screen.value.rotation.x = innerDisplay.value.rotation.x;
  }

  // Update controls if used
  // controls.value?.update();

  // Render
  if (scene.value && camera.value) {
    renderer.value.render(scene.value, camera.value);
  }
};

// --- Lifecycle Hooks ---
onMounted(() => {
  initThree();
  // Note: animate() is called inside the loader's success callback now
});

onUnmounted(() => {
  // Cancel animation loop
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId);
  }
  // Remove resize listener
  window.removeEventListener('resize', handleResize);

  // Dispose of Three.js resources
  if (renderer.value) {
    renderer.value.dispose();
  }
  if (scene.value) {
     // Basic cleanup: Traverse and dispose geometries/materials
     scene.value.traverse((object) => {
       if (object instanceof THREE.Mesh) {
         if (object.geometry) {
           object.geometry.dispose();
         }
         if (Array.isArray(object.material)) {
           object.material.forEach(material => material.dispose());
         } else if (object.material) {
           object.material.dispose();
         }
       }
     });
  }
  // Optional: Dispose controls
  // controls.value?.dispose();


  // Clear refs
  scene.value = null;
  camera.value = null;
  renderer.value = null;
  innerDisplay.value = null;
  outerDisplay.value = null;
  screen.value = null;
  // controls.value = null;
  canvasRef.value = null; // Not strictly necessary but good practice

  console.log("Three.js resources cleaned up");
});

</script>

<template>
  <canvas ref="canvasRef" class="webgl-canvas"></canvas>

  </template>

<style scoped>
/* Keep relevant styles, or remove them if removing default content */
/* .logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}
.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
} */

/* Ensure canvas takes up space */
.webgl-canvas {
  display: block; /* Prevent extra space below canvas */
  width: 100vw;   /* Take full viewport width */
  height: 100vh;  /* Take full viewport height */
  position: fixed; /* Position relative to viewport */
  top: 0;
  left: 0;
  z-index: -1; /* Place behind other content if necessary */
}
</style>