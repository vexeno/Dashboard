<script setup>
import { computed, onMounted, onUnmounted, ref } from 'vue'

const defaultConfig = {
  configVersion: 5,
  siteTitle: 'SeeHex',
  domain: 'seehex.com',
  name: 'SeeHex',
  headlinePrefix: 'This is',
  avatarDark: '/avatar-dark.png',
  avatarLight: '/avatar-light.png',
  status: 'online',
  typingTexts: ['LESS IS MORE'],
  contacts: ['x@seehex.com'],
  github: 'https://github.com/vexeno',
  cards: [
    {
      label: 'ImgBed',
      href: 'https://img.seehex.com',
      icon: 'fa-regular fa-images',
      color: '#a78bfa',
    },
    {
      label: 'Domains',
      href: 'https://mi.seehex.com',
      icon: 'fa-solid fa-globe',
      color: '#facc15',
    },
    {
      label: 'Status',
      href: 'https://status.seehex.com',
      icon: 'fa-solid fa-signal',
      color: '#22c55e',
    },
    {
      label: 'Contact',
      href: 'https://t.me/vyoss',
      icon: 'fa-regular fa-paper-plane',
      color: '#38bdf8',
    },
  ],
}

const typingSpeed = 125
const storageKey = 'seehex-home-config'

const config = ref({ ...defaultConfig })
const isDark = ref(true)
const showEmail = ref(false)
const showEditor = ref(new URLSearchParams(window.location.search).has('edit'))
const canEdit = showEditor.value
const activeCard = ref(null)
const cardHoverTimers = new Map()
const elapsedSeconds = ref(0)
const typedText = ref('')
const clickSparkCanvas = ref(null)
const editorText = ref('')
const editorError = ref('')
let timerId
let typingTimerId
let typingCharIndex = 0
let clickSparkAnimationId
const clickSparks = []
const clickSparkOptions = {
  color: '#facc15',
  size: 10,
  radius: 20,
  count: 8,
  duration: 420,
  extraScale: 1,
}

const typingTexts = computed(() => {
  const texts = config.value.typingTexts?.filter(Boolean)
  return texts?.length ? texts : defaultConfig.typingTexts
})

const currentAvatar = computed(() => (isDark.value ? config.value.avatarDark : config.value.avatarLight))


const splitSegmenter = typeof Intl !== 'undefined' && Intl.Segmenter
  ? new Intl.Segmenter(undefined, { granularity: 'grapheme' })
  : null

function splitTextChars(text) {
  const value = String(text || '')
  const chars = splitSegmenter
    ? Array.from(splitSegmenter.segment(value), (part) => part.segment)
    : Array.from(value)

  return chars.map((char) => ({
    char,
    display: char === ' ' ? '\u00a0' : char,
  }))
}

const hiChars = computed(() => splitTextChars('Hi'))
const headlinePrefixChars = computed(() => splitTextChars(config.value.headlinePrefix))
const headlineNameChars = computed(() => splitTextChars(config.value.name))
const headlineLabel = computed(() => [config.value.headlinePrefix, config.value.name].filter(Boolean).join(' '))

const formattedTime = computed(() => {
  const hours = String(Math.floor(elapsedSeconds.value / 3600)).padStart(2, '0')
  const minutes = String(Math.floor((elapsedSeconds.value % 3600) / 60)).padStart(2, '0')
  const seconds = String(elapsedSeconds.value % 60).padStart(2, '0')

  return { hours, minutes, seconds }
})

async function loadConfig() {
  try {
    const response = await fetch('/site.config.json', { cache: 'no-store' })
    const remoteConfig = response.ok ? await response.json() : {}
    const savedConfig = JSON.parse(window.localStorage.getItem(storageKey) || 'null')
    const shouldUseSavedConfig = savedConfig?.configVersion === remoteConfig?.configVersion
    config.value = normalizeConfig(shouldUseSavedConfig ? savedConfig : remoteConfig)
    document.title = config.value.siteTitle || defaultConfig.siteTitle
  } catch {
    config.value = normalizeConfig(defaultConfig)
  }

  syncEditorText()
}

function normalizeConfig(nextConfig) {
  return {
    ...defaultConfig,
    ...nextConfig,
    avatarDark: nextConfig?.avatarDark || nextConfig?.avatar || defaultConfig.avatarDark,
    avatarLight: nextConfig?.avatarLight || nextConfig?.avatar || defaultConfig.avatarLight,
    typingTexts: Array.isArray(nextConfig?.typingTexts) ? nextConfig.typingTexts : defaultConfig.typingTexts,
    contacts: Array.isArray(nextConfig?.contacts) ? nextConfig.contacts : defaultConfig.contacts,
    cards: Array.isArray(nextConfig?.cards) ? nextConfig.cards : defaultConfig.cards,
  }
}

function toggleTheme() {
  isDark.value = !isDark.value
}

function toggleEmail() {
  showEmail.value = !showEmail.value
}

function queueCardFill(cardKey) {
  clearCardTimer(cardKey)
  cardHoverTimers.set(cardKey, window.setTimeout(() => {
    activeCard.value = cardKey
    cardHoverTimers.delete(cardKey)
  }, 180))
}

function clearCardFill(cardKey) {
  clearCardTimer(cardKey)

  if (activeCard.value === cardKey) {
    activeCard.value = null
  }
}

function clearCardTimer(cardKey) {
  const hoverTimer = cardHoverTimers.get(cardKey)

  if (hoverTimer) {
    window.clearTimeout(hoverTimer)
    cardHoverTimers.delete(cardKey)
  }
}

function openEditor() {
  syncEditorText()
  showEditor.value = true
}

function closeEditor() {
  showEditor.value = false
  editorError.value = ''
}

function syncEditorText() {
  editorText.value = JSON.stringify(config.value, null, 2)
}

function applyEditorConfig() {
  try {
    const parsedConfig = JSON.parse(editorText.value)
    config.value = normalizeConfig(parsedConfig)
    window.localStorage.setItem(storageKey, JSON.stringify(config.value))
    document.title = config.value.siteTitle || defaultConfig.siteTitle
    editorError.value = ''
    resetTyping()
  } catch (error) {
    editorError.value = `JSON 格式错误：${error.message}`
  }
}

function resetEditorConfig() {
  window.localStorage.removeItem(storageKey)
  loadConfig()
}

function downloadConfig() {
  const blob = new Blob([JSON.stringify(config.value, null, 2)], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  link.href = url
  link.download = 'site.config.json'
  link.click()
  URL.revokeObjectURL(url)
}

function resetTyping() {
  window.clearTimeout(typingTimerId)
  stopClickSpark()
  typingCharIndex = 0
  typedText.value = ''
  typeNextFrame()
}

function typeNextFrame() {
  const currentText = typingTexts.value[0] || ''

  if (typingCharIndex >= currentText.length) {
    typedText.value = currentText
    return
  }

  typingCharIndex += 1
  typedText.value = currentText.slice(0, typingCharIndex)
  typingTimerId = window.setTimeout(typeNextFrame, typingSpeed)
}

function resizeClickSparkCanvas() {
  const canvas = clickSparkCanvas.value
  if (!canvas) return

  const ratio = Math.min(window.devicePixelRatio || 1, 2)
  const width = window.innerWidth
  const height = window.innerHeight
  canvas.width = Math.floor(width * ratio)
  canvas.height = Math.floor(height * ratio)
  canvas.style.width = width + 'px'
  canvas.style.height = height + 'px'

  const ctx = canvas.getContext('2d')
  if (ctx) {
    ctx.setTransform(ratio, 0, 0, ratio, 0, 0)
  }
}

function clickSparkEase(progress) {
  return progress * (2 - progress)
}

function drawClickSparks(timestamp) {
  const canvas = clickSparkCanvas.value
  const ctx = canvas?.getContext('2d')
  if (!canvas || !ctx) return

  ctx.clearRect(0, 0, window.innerWidth, window.innerHeight)

  for (let index = clickSparks.length - 1; index >= 0; index -= 1) {
    const spark = clickSparks[index]
    const elapsed = timestamp - spark.startTime

    if (elapsed >= clickSparkOptions.duration) {
      clickSparks.splice(index, 1)
      continue
    }

    const progress = elapsed / clickSparkOptions.duration
    const eased = clickSparkEase(progress)
    const distance = eased * clickSparkOptions.radius * clickSparkOptions.extraScale
    const lineLength = clickSparkOptions.size * (1 - eased)
    const x1 = spark.x + distance * Math.cos(spark.angle)
    const y1 = spark.y + distance * Math.sin(spark.angle)
    const x2 = spark.x + (distance + lineLength) * Math.cos(spark.angle)
    const y2 = spark.y + (distance + lineLength) * Math.sin(spark.angle)

    ctx.strokeStyle = clickSparkOptions.color
    ctx.lineWidth = 2
    ctx.lineCap = 'round'
    ctx.beginPath()
    ctx.moveTo(x1, y1)
    ctx.lineTo(x2, y2)
    ctx.stroke()
  }

  clickSparkAnimationId = window.requestAnimationFrame(drawClickSparks)
}

function handleClickSpark(event) {
  if (window.matchMedia('(prefers-reduced-motion: reduce)').matches) return
  const now = performance.now()
  for (let index = 0; index < clickSparkOptions.count; index += 1) {
    clickSparks.push({
      x: event.clientX,
      y: event.clientY,
      angle: (Math.PI * 2 * index) / clickSparkOptions.count,
      startTime: now,
    })
  }
}

function startClickSpark() {
  resizeClickSparkCanvas()
  window.addEventListener('resize', resizeClickSparkCanvas)
  window.addEventListener('click', handleClickSpark, true)
  clickSparkAnimationId = window.requestAnimationFrame(drawClickSparks)
}

function stopClickSpark() {
  window.removeEventListener('resize', resizeClickSparkCanvas)
  window.removeEventListener('click', handleClickSpark, true)
  if (clickSparkAnimationId) {
    window.cancelAnimationFrame(clickSparkAnimationId)
  }
  clickSparks.length = 0
}

onMounted(async () => {
  await loadConfig()
  typeNextFrame()
  startClickSpark()

  timerId = window.setInterval(() => {
    elapsedSeconds.value += 1
  }, 1000)
})

onUnmounted(() => {
  window.clearInterval(timerId)
  window.clearTimeout(typingTimerId)
  cardHoverTimers.forEach((hoverTimer) => window.clearTimeout(hoverTimer))
  cardHoverTimers.clear()
})
</script>

<template>
  <main class="page" :class="{ 'dark-mode': isDark }">
    <canvas ref="clickSparkCanvas" class="click-spark-canvas" aria-hidden="true"></canvas>
    <section class="hero" aria-label="个人导航页">
      <div class="profile">
        <div class="avatar-wrap">
          <img class="avatar" :src="currentAvatar" alt="头像" />
        </div>

        <div class="intro">
          <h1 class="title-hi seehex-split-title" aria-label="Hi">
            <span aria-hidden="true" class="seehex-split-line">
              <span
                v-for="(part, index) in hiChars"
                :key="'hi-' + index + '-' + part.char"
                class="seehex-split-char"
                :style="{ '--i': index, '--split-base': 0 }"
              >{{ part.display }}</span>
            </span>
          </h1>
          <h1 class="title-main seehex-split-title" :aria-label="headlineLabel">
            <span aria-hidden="true" class="seehex-split-line">
              <span
                v-for="(part, index) in headlinePrefixChars"
                :key="'prefix-' + index + '-' + part.char"
                class="seehex-split-char"
                :style="{ '--i': index, '--split-base': 5 }"
              >{{ part.display }}</span>
              <span class="seehex-split-char" :style="{ '--i': headlinePrefixChars.length, '--split-base': 5 }">&nbsp;</span>
              <span class="name-style">
                <span
                  v-for="(part, index) in headlineNameChars"
                  :key="'name-' + index + '-' + part.char"
                  class="seehex-split-char"
                  :style="{ '--i': headlinePrefixChars.length + 1 + index, '--split-base': 5 }"
                >{{ part.display }}</span>
              </span>
            </span>
          </h1>
          <p class="quote" aria-label="签名">
            <span class="quote-mark">“</span>
            <span class="typing-wrap"><span class="typing">{{ typedText }}</span><span class="caret"></span></span>
            <span class="quote-mark">”</span>
          </p>
        </div>
      </div>

      <div class="actions" aria-label="快捷操作">
        <div class="action-holder">
          <button class="action-button" type="button" @click="toggleEmail">
            <i class="fa-solid fa-envelope"></i>
            <span>Email</span>
          </button>
          <Transition name="float">
            <div v-if="showEmail" class="email-popover">
              <a v-for="email in config.contacts" :key="email" :href="`mailto:${email}`">{{ email }}</a>
            </div>
          </Transition>
        </div>

        <a class="action-button" :href="config.github" target="_blank" rel="noreferrer">
          <i class="fa-brands fa-github"></i>
          <span>Github</span>
        </a>

        <button v-if="canEdit" class="action-button" type="button" @click="openEditor">
          <i class="fa-solid fa-sliders"></i>
          <span>编辑</span>
        </button>

        <button class="action-button" type="button" @click="toggleTheme">
          <i :class="isDark ? 'fa-solid fa-moon' : 'fa-solid fa-sun'"></i>
          <span>{{ isDark ? 'Dark' : 'Light' }}</span>
        </button>
      </div>

      <nav class="site-grid" aria-label="站点导航">
        <a
          v-for="card in config.cards"
          :key="`${card.label}-${card.href}`"
          class="site-card"
          :class="{ 'is-fill-active': activeCard === `${card.label}-${card.href}` }"
          :href="card.href"
          :target="card.href?.startsWith('http') ? '_blank' : undefined"
          rel="noreferrer"
          :style="{ '--card-accent': card.color || '#facc15' }"
          @mouseenter="queueCardFill(`${card.label}-${card.href}`)"
          @mouseleave="clearCardFill(`${card.label}-${card.href}`)"
          @focus="queueCardFill(`${card.label}-${card.href}`)"
          @blur="clearCardFill(`${card.label}-${card.href}`)"
        >
          <i :class="card.icon || 'fa-solid fa-link'"></i>
          <span>{{ card.label }}</span>
        </a>
      </nav>
    </section>

    <div class="visit-timer" aria-label="停留时间">
      <i class="fa-regular fa-clock"></i>
      <span>uptime :</span>
      <strong>{{ formattedTime.hours }}</strong>
      <span>:</span>
      <strong>{{ formattedTime.minutes }}</strong>
      <span>:</span>
      <strong>{{ formattedTime.seconds }}</strong>
    </div>

    <Transition name="fade">
      <div v-if="showEditor" class="editor-mask" @click.self="closeEditor">
        <section class="editor-panel" aria-label="首页配置编辑器">
          <div class="editor-head">
            <div>
              <p class="editor-kicker">site.config.json</p>
              <h2>首页配置</h2>
            </div>
            <button class="icon-button" type="button" aria-label="关闭" @click="closeEditor">
              <i class="fa-solid fa-xmark"></i>
            </button>
          </div>

          <p class="editor-help">
            修改下面 JSON 后点“应用到页面”。满意后点“下载配置”，用下载的文件替换
            <code>public/site.config.json</code>。编辑入口只在 URL 带 <code>?edit=1</code> 时显示，线上访客默认看不到。
          </p>

          <textarea v-model="editorText" spellcheck="false" class="editor-textarea"></textarea>
          <p v-if="editorError" class="editor-error">{{ editorError }}</p>

          <div class="editor-actions">
            <button class="action-button" type="button" @click="applyEditorConfig">应用到页面</button>
            <button class="action-button" type="button" @click="downloadConfig">下载配置</button>
            <button class="action-button muted-button" type="button" @click="resetEditorConfig">恢复文件默认</button>
          </div>
        </section>
      </div>
    </Transition>
  </main>
</template>
