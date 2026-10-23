<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'
import chairUrl from '../assets/chair.glb'

const viewer = ref<HTMLDivElement | null>(null)
const loading = ref(true)
const loadError = ref(false)
let renderer: THREE.WebGLRenderer | undefined
let controls: OrbitControls | undefined
let animationFrame = 0
let resizeObserver: ResizeObserver | undefined

onMounted(() => {
  if (!viewer.value) return
  const container = viewer.value
  const scene = new THREE.Scene()
  scene.background = new THREE.Color(0xffffff)
  const camera = new THREE.PerspectiveCamera(32, 1, 0.01, 100)
  camera.position.set(3.2, 2.1, 4.2)

  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.setSize(container.clientWidth, container.clientHeight)
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
  renderer.outputColorSpace = THREE.SRGBColorSpace
  renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMappingExposure = 1.1
  container.appendChild(renderer.domElement)

  scene.add(new THREE.HemisphereLight(0xffffff, 0xe8e8e8, 2.4))
  const keyLight = new THREE.DirectionalLight(0xffffff, 3.2)
  keyLight.position.set(4, 6, 3)
  keyLight.castShadow = true
  keyLight.shadow.mapSize.set(2048, 2048)
  scene.add(keyLight)
  const fillLight = new THREE.DirectionalLight(0xdcecff, 1.2)
  fillLight.position.set(-4, 2, -3)
  scene.add(fillLight)

  const floor = new THREE.Mesh(
    new THREE.CircleGeometry(5, 64),
    new THREE.ShadowMaterial({ color: 0x6b7280, opacity: 0.16 }),
  )
  floor.rotation.x = -Math.PI / 2
  floor.position.y = -0.02
  floor.receiveShadow = true
  scene.add(floor)

  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.06
  controls.minDistance = 1.2
  controls.maxDistance = 8
  controls.minPolarAngle = 0.3
  controls.maxPolarAngle = Math.PI / 2 + 0.15
  controls.target.set(0, 0.8, 0)

  new GLTFLoader().load(chairUrl, (gltf) => {
    const model = gltf.scene
    const bounds = new THREE.Box3().setFromObject(model)
    const size = bounds.getSize(new THREE.Vector3())
    const center = bounds.getCenter(new THREE.Vector3())
    const scale = 2.2 / Math.max(size.x, size.y, size.z)
    model.scale.setScalar(scale)
    model.position.set(-center.x * scale, -bounds.min.y * scale, -center.z * scale)
    model.traverse((object) => {
      if (object instanceof THREE.Mesh) {
        object.castShadow = true
        object.receiveShadow = true
      }
    })
    scene.add(model)
    controls?.target.set(0, size.y * scale * 0.45, 0)
    controls?.update()
    loading.value = false
  }, undefined, () => {
    loading.value = false
    loadError.value = true
  })

  const resize = () => {
    if (!renderer || !viewer.value) return
    const { clientWidth, clientHeight } = viewer.value
    camera.aspect = clientWidth / clientHeight
    camera.updateProjectionMatrix()
    renderer.setSize(clientWidth, clientHeight)
  }
  resizeObserver = new ResizeObserver(resize)
  resizeObserver.observe(container)
  resize()
  const animate = () => {
    animationFrame = requestAnimationFrame(animate)
    controls?.update()
    renderer?.render(scene, camera)
  }
  animate()
})

onBeforeUnmount(() => {
  cancelAnimationFrame(animationFrame)
  resizeObserver?.disconnect()
  controls?.dispose()
  renderer?.dispose()
})
</script>

<template>
  <main class="page-shell">
    <header class="page-header">
      <p class="eyebrow">LUU HUU DUOC DEMO</p>
      <h1>demo hiệu ứng 3D</h1>
      <p class="hint">Kéo để xoay · Cuộn để phóng to / thu nhỏ</p>
    </header>
    <section ref="viewer" class="model-viewer" aria-label="Trình xem model 3D">
      <div v-if="loading" class="status">Đang tải model…</div>
      <div v-else-if="loadError" class="status error">Không thể tải `chair.glb`.</div>
    </section>
  </main>
</template>

<style scoped>
.page-shell { width: min(1180px, 100%); min-height: 100vh; margin: 0 auto; padding: 52px 28px 28px; color: #141414; }
.page-header { padding: 0 4px 22px; }
.eyebrow { margin: 0 0 8px; color: #8b8b8b; font-size: 11px; font-weight: 700; letter-spacing: 0.2em; }
h1 { margin: 0; font-size: clamp(32px, 5vw, 54px); font-weight: 700; letter-spacing: -0.05em; line-height: 1; }
.hint { margin: 14px 0 0; color: #888; font-size: 13px; }
.model-viewer { position: relative; width: 100%; height: min(72vh, 700px); min-height: 430px; overflow: hidden; border: 1px solid #ededed; border-radius: 24px; background: #fff; box-shadow: 0 18px 50px rgba(0, 0, 0, 0.05); cursor: grab; }
.model-viewer:active { cursor: grabbing; }
.model-viewer canvas { display: block; width: 100%; height: 100%; }
.status { position: absolute; inset: 50% auto auto 50%; z-index: 1; transform: translate(-50%, -50%); color: #777; font-size: 13px; pointer-events: none; }
.error { color: #b42318; }
@media (max-width: 600px) { .page-shell { padding: 30px 14px 14px; } .model-viewer { min-height: 430px; border-radius: 18px; } }
</style>
