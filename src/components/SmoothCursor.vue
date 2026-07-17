<script setup>
import { onMounted, onUnmounted, reactive, ref } from 'vue'

const DESKTOP_POINTER_QUERY = '(any-hover: hover) and (any-pointer: fine)'
const springConfig = { damping: 45, stiffness: 400, mass: 1, restDelta: 0.001 }
const rotationSpringConfig = { ...springConfig, damping: 60, stiffness: 300 }
const scaleSpringConfig = { ...springConfig, stiffness: 500, damping: 35 }

function isTrackablePointer(pointerType) { return pointerType !== 'touch' }
function createSpring(initialValue, config) {
  return { value: initialValue, target: initialValue, velocity: 0, config }
}

function stepSpring(spring, deltaTime) {
  const dt = Math.min(deltaTime, 0.064)
  const displacement = spring.value - spring.target
  const springForce = -spring.config.stiffness * displacement
  const dampingForce = -spring.config.damping * spring.velocity
  const acceleration = (springForce + dampingForce) / spring.config.mass
  spring.velocity += acceleration * dt
  spring.value += spring.velocity * dt
  if (Math.abs(spring.velocity) < spring.config.restDelta && Math.abs(displacement) < spring.config.restDelta) {
    spring.value = spring.target
    spring.velocity = 0
  }
}

const isEnabled = ref(false)
const isVisible = ref(false)
const cursor = reactive({
  x: createSpring(0, springConfig),
  y: createSpring(0, springConfig),
  rotation: createSpring(0, rotationSpringConfig),
  scale: createSpring(1, scaleSpringConfig),
})

const lastMousePos = { x: 0, y: 0 }
const velocity = { x: 0, y: 0 }
let lastUpdateTime = Date.now()
let previousAngle = 0
let accumulatedRotation = 0
let mediaQuery
let rafId = 0
let springRafId = 0
let scaleTimeout = null
let lastFrameTime = performance.now()
let previousBodyCursor = ''

function updateVelocity(currentPos) {
  const currentTime = Date.now()
  const deltaTime = currentTime - lastUpdateTime
  if (deltaTime > 0) {
    velocity.x = (currentPos.x - lastMousePos.x) / deltaTime
    velocity.y = (currentPos.y - lastMousePos.y) / deltaTime
  }
  lastUpdateTime = currentTime
  lastMousePos.x = currentPos.x
  lastMousePos.y = currentPos.y
}

function smoothPointerMove(e) {
  if (!isTrackablePointer(e.pointerType)) return
  isVisible.value = true
  const currentPos = { x: e.clientX, y: e.clientY }
  updateVelocity(currentPos)
  const speed = Math.sqrt(Math.pow(velocity.x, 2) + Math.pow(velocity.y, 2))
  cursor.x.target = currentPos.x
  cursor.y.target = currentPos.y
  if (speed > 0.1) {
    const currentAngle = Math.atan2(velocity.y, velocity.x) * (180 / Math.PI) + 90
    let angleDiff = currentAngle - previousAngle
    if (angleDiff > 180) angleDiff -= 360
    if (angleDiff < -180) angleDiff += 360
    accumulatedRotation += angleDiff
    cursor.rotation.target = accumulatedRotation
    previousAngle = currentAngle
    cursor.scale.target = 0.95
    if (scaleTimeout !== null) clearTimeout(scaleTimeout)
    scaleTimeout = setTimeout(() => { cursor.scale.target = 1 }, 150)
  }
}

function throttledPointerMove(e) {
  if (!isTrackablePointer(e.pointerType)) return
  if (rafId) return
  rafId = requestAnimationFrame(() => {
    smoothPointerMove(e)
    rafId = 0
  })
}

function animateSprings(now = performance.now()) {
  const deltaTime = (now - lastFrameTime) / 1000
  lastFrameTime = now
  stepSpring(cursor.x, deltaTime)
  stepSpring(cursor.y, deltaTime)
  stepSpring(cursor.rotation, deltaTime)
  stepSpring(cursor.scale, deltaTime)
  springRafId = requestAnimationFrame(animateSprings)
}

function enableCursor() {
  if (isEnabled.value) return
  isEnabled.value = true
  isVisible.value = false
  previousBodyCursor = document.body.style.cursor
  document.body.style.cursor = 'none'
  lastFrameTime = performance.now()
  window.addEventListener('pointermove', throttledPointerMove, { passive: true })
  springRafId = requestAnimationFrame(animateSprings)
}

function disableCursor() {
  if (!isEnabled.value) return
  isEnabled.value = false
  isVisible.value = false
  window.removeEventListener('pointermove', throttledPointerMove)
  document.body.style.cursor = previousBodyCursor || 'auto'
  if (rafId) cancelAnimationFrame(rafId)
  if (springRafId) cancelAnimationFrame(springRafId)
  if (scaleTimeout !== null) clearTimeout(scaleTimeout)
  rafId = 0
  springRafId = 0
  scaleTimeout = null
}

function updateEnabled() {
  if (mediaQuery?.matches) enableCursor()
  else disableCursor()
}

onMounted(() => {
  mediaQuery = window.matchMedia(DESKTOP_POINTER_QUERY)
  updateEnabled()
  mediaQuery.addEventListener('change', updateEnabled)
})

onUnmounted(() => {
  mediaQuery?.removeEventListener('change', updateEnabled)
  disableCursor()
})
</script>

<template>
  <div
    v-if="isEnabled"
    class="smooth-cursor"
    :style="{
      left: cursor.x.value + 'px',
      top: cursor.y.value + 'px',
      transform: 'translate(-50%, -50%) rotate(' + cursor.rotation.value + 'deg) scale(' + cursor.scale.value + ')',
      opacity: isVisible ? 1 : 0,
    }"
    aria-hidden="true"
  >
    <svg xmlns="http://www.w3.org/2000/svg" width="50" height="54" viewBox="0 0 50 54" fill="none" style="scale: 0.5">
      <path
        d="M42.6817 41.1495L27.5103 6.79925C26.7269 5.02557 24.2082 5.02558 23.3927 6.79925L7.59814 41.1495C6.75833 42.9759 8.52712 44.8902 10.4125 44.1954L24.3757 39.0496C24.8829 38.8627 25.4385 38.8627 25.9422 39.0496L39.8121 44.1954C41.6849 44.8902 43.4884 42.9759 42.6817 41.1495Z"
        fill="black"
      />
      <path
        d="M43.7146 40.6933L28.5431 6.34306C27.3556 3.65428 23.5772 3.69516 22.3668 6.32755L6.57226 40.6778C5.3134 43.4156 7.97238 46.298 10.803 45.2549L24.7662 40.109C25.0221 40.0147 25.2999 40.0156 25.5494 40.1082L39.4193 45.254C42.2261 46.2953 44.9254 43.4347 43.7146 40.6933Z"
        stroke="white"
        stroke-width="2.25825"
      />
    </svg>
  </div>
</template>

<style scoped>
.smooth-cursor {
  position: fixed;
  z-index: 100;
  pointer-events: none;
  will-change: transform;
  transition: opacity 0.15s ease;
  filter: drop-shadow(0 2.25825px 2.25825px rgba(0, 0, 0, 0.08));
}
</style>
