<script setup lang="ts">
import { ref, reactive, computed, onMounted, onUnmounted } from 'vue'

// --- State Management ---
const isAudioInitialized = ref(false)
const masterVolume = ref(0.3)
const activeEffect = ref('Ambient Dream')
const lastNotePlayed = ref('None')
const stats = reactive({
  totalClicks: 0,
  maxVelocity: 0,
  currentX: 0,
  currentY: 0,
})

// Particle array for the canvas-like CSS grid
interface Particle {
  id: number
  x: number
  y: number
  size: number
  color: string
  alpha: number
  scale: number
}
const particles = ref<Particle[]>([])
let particleId = 0

// Track mouse speed/velocity
let lastMouseTime = performance.now()
let lastMouseX = 0
let lastMouseY = 0
const mouseVelocity = ref(0)

// --- Web Audio Synthesizer Logic ---
let audioCtx: AudioContext | null = null
let masterGain: GainNode | null = null

const initAudio = () => {
  if (isAudioInitialized.value) return
  audioCtx = new (window.AudioContext || (window as any).webkitAudioContext)()
  masterGain = audioCtx.createGain()
  masterGain.gain.setValueAtTime(masterVolume.value, audioCtx.currentTime)
  masterGain.connect(audioCtx.destination)
  isAudioInitialized.value = true
}

// Update volume reactively
const updateVolume = () => {
  if (masterGain && audioCtx) {
    masterGain.gain.linearRampToValueAtTime(masterVolume.value, audioCtx.currentTime + 0.05)
  }
}

// Play a mathematical synth frequency based on interactions
const playSynthNote = (
  frequency: number,
  type: 'sine' | 'square' | 'triangle' | 'sawtooth' = 'sine',
  duration = 0.4,
) => {
  if (!audioCtx || !masterGain) return

  const osc = audioCtx.createOscillator()
  const gainNode = audioCtx.createGain()

  osc.type = type
  osc.frequency.setValueAtTime(frequency, audioCtx.currentTime)

  // High velocity = sliding pitch shift
  if (mouseVelocity.value > 1.5) {
    osc.frequency.exponentialRampToValueAtTime(frequency * 2, audioCtx.currentTime + duration)
  }

  // Smooth audio envelope (Prevents clicking sounds)
  gainNode.gain.setValueAtTime(0, audioCtx.currentTime)
  gainNode.gain.linearRampToValueAtTime(0.4, audioCtx.currentTime + 0.05)
  gainNode.gain.exponentialRampToValueAtTime(0.0001, audioCtx.currentTime + duration)

  osc.connect(gainNode)
  gainNode.connect(masterGain)

  osc.start()
  osc.stop(audioCtx.currentTime + duration)

  lastNotePlayed.value = `${Math.round(frequency)}Hz (${type})`
}

// --- Interactive Events & Generators ---
const handleGlobalMouseMove = (e: MouseEvent) => {
  stats.currentX = e.clientX
  stats.currentY = e.clientY

  // Calculate mouse velocity
  const now = performance.now()
  const dt = now - lastMouseTime
  if (dt > 0) {
    const dx = e.clientX - lastMouseX
    const dy = e.clientY - lastMouseY
    const dist = Math.sqrt(dx * dx + dy * dy)
    mouseVelocity.value = dist / dt
    if (mouseVelocity.value > stats.maxVelocity) {
      stats.maxVelocity = parseFloat(mouseVelocity.value.toFixed(2))
    }
  }
  lastMouseX = e.clientX
  lastMouseY = e.clientY
  lastMouseTime = now

  // Spawning passive reactive environment particles
  if (Math.random() < 0.15) {
    spawnParticle(e.clientX, e.clientY)
  }
}

const handleBoardClick = (e: MouseEvent) => {
  stats.totalClicks++
  initAudio() // Fallback safe init on first interaction

  // Spark 15 burst particles
  for (let i = 0; i < 15; i++) {
    spawnParticle(e.clientX, e.clientY, true)
  }

  // Synthesize chord dynamic mapping to coordinate clicks
  if (isAudioInitialized.value) {
    const freqBase = 150 + (e.clientX % 400)
    if (activeEffect.value === 'Ambient Dream') {
      playSynthNote(freqBase, 'sine', 0.8)
      playSynthNote(freqBase * 1.5, 'triangle', 0.5)
    } else if (activeEffect.value === 'Cyber Chiptune') {
      playSynthNote(freqBase * 2, 'square', 0.2)
      setTimeout(() => playSynthNote(freqBase * 2.5, 'square', 0.15), 60)
    } else {
      playSynthNote(freqBase * 0.7, 'sawtooth', 0.4)
    }
  }
}

const spawnParticle = (x: number, y: number, isBurst = false) => {
  const colors =
    activeEffect.value === 'Ambient Dream'
      ? ['#42b883', '#35495e', '#64748b', '#00dc82']
      : activeEffect.value === 'Cyber Chiptune'
        ? ['#ff007f', '#00f0ff', '#ffb700', '#ffffff']
        : ['#ff4500', '#ff8c00', '#ff0000', '#4a00e0']

  const p: Particle = {
    id: particleId++,
    x: x + (isBurst ? (Math.random() - 0.5) * 160 : 0),
    y: y + (isBurst ? (Math.random() - 0.5) * 160 : 0),
    size: Math.random() * (isBurst ? 14 : 6) + 4,
    color: colors[Math.floor(Math.random() * colors.length)],
    alpha: 1,
    scale: 1,
  }
  particles.value.push(p)

  // Gentle particle decay animation
  setTimeout(() => {
    particles.value = particles.value.filter((item) => item.id !== p.id)
  }, 1200)
}

// --- Dynamic Color Styles Matrix ---
const themeColor = computed(() => {
  if (activeEffect.value === 'Ambient Dream') return '#42b883'
  if (activeEffect.value === 'Cyber Chiptune') return '#00f0ff'
  return '#ff4500'
})

// Hook up event loops cleanly
onMounted(() => {
  window.addEventListener('mousemove', handleGlobalMouseMove)
})
onUnmounted(() => {
  window.removeEventListener('mousemove', handleGlobalMouseMove)
  if (audioCtx) audioCtx.close()
})
</script>

<template>
  <div class="matrix-wrapper" @click="handleBoardClick">
    <!-- Generative Floating Elements Layer -->
    <div
      v-for="p in particles"
      :key="p.id"
      class="quantum-particle"
      :style="{
        left: p.x + 'px',
        top: p.y + 'px',
        width: p.size + 'px',
        height: p.size + 'px',
        backgroundColor: p.color,
        boxShadow: `0 0 15px ${p.color}`,
      }"
    ></div>

    <!-- Interface Dashboard Block Container -->
    <div class="glass-dashboard" @click.stop>
      <header class="branding">
        <div class="logo-node">
          <span class="pulse-dot" :style="{ backgroundColor: themeColor }"></span>
          Vue<span :style="{ color: themeColor }">AudioSynthetics</span>
        </div>
        <p>A multi-sensory interactive interface showing reactive capability.</p>
      </header>

      <hr class="divider" />

      <!-- Step 1 Audio Permission Activation Toggle Box -->
      <div v-if="!isAudioInitialized" class="activation-banner" @click="initAudio">
        <div class="icon-pulse">🔊</div>
        <h3>Click here to Unlock Audio Engines</h3>
        <p>Uses native Web Audio API oscillators directly synchronized into the system core.</p>
      </div>

      <!-- Controls Dashboard Workspace Grid -->
      <section class="controls-matrix">
        <div class="control-card">
          <label>Oscillator Blueprint</label>
          <div class="selector-grid">
            <button
              v-for="mode in ['Ambient Dream', 'Cyber Chiptune', 'Solar Plasma']"
              :key="mode"
              :class="{ active: activeEffect === mode }"
              @click="activeEffect = mode"
              :style="activeEffect === mode ? { borderColor: themeColor, color: themeColor } : {}"
            >
              {{ mode }}
            </button>
          </div>
        </div>

        <div class="control-card">
          <label>Master Dynamics Gain Volume: {{ Math.round(masterVolume * 100) }}%</label>
          <input
            type="range"
            min="0"
            max="0.8"
            step="0.01"
            v-model="masterVolume"
            @input="updateVolume"
            class="neon-slider"
            :style="{ '--thumb-color': themeColor }"
          />
        </div>
      </section>

      <!-- Realtime Analytical Data Panel -->
      <section class="telemetry-deck">
        <div class="metric">
          <span class="label">Total Impacts</span>
          <span class="val text-glow">{{ stats.totalClicks }}</span>
        </div>
        <div class="metric">
          <span class="label">Cursor Max Velocity</span>
          <span class="val">{{ stats.maxVelocity }} px/ms</span>
        </div>
        <div class="metric">
          <span class="label">Oscillator Vector</span>
          <span class="val" :style="{ color: themeColor }">{{ lastNotePlayed }}</span>
        </div>
        <div class="metric">
          <span class="label">Coordinate Grid</span>
          <span class="val text-dim">X: {{ stats.currentX }} | Y: {{ stats.currentY }}</span>
        </div>
      </section>

      <!-- Interactive Playground Help Pad -->
      <div class="instruction-pad" :style="{ borderColor: themeColor + '33' }">
        💡 <strong>Click anywhere on the open background</strong> to spawn explosive graphic bursts
        and dynamically map coordinate-to-pitch soundscapes. Swipe your mouse rapidly to bend
        frequencies!
      </div>
    </div>
  </div>
</template>

<style scoped>
/* --- Design Architecture & Variables --- */
.matrix-wrapper {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: radial-gradient(circle at 50% 50%, #0b0f19 0%, #030508 100%);
  color: #e2e8f0;
  font-family:
    'SF Pro Display',
    system-ui,
    -apple-system,
    sans-serif;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
  user-select: none;
}

/* Glassmorphism Panel Dashboard UI */
.glass-dashboard {
  background: rgba(15, 23, 42, 0.65);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 24px;
  padding: 2.5rem;
  width: 90%;
  max-width: 640px;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.5);
  z-index: 10;
  position: relative;
}

.branding .logo-node {
  font-size: 1.8rem;
  font-weight: 800;
  letter-spacing: -0.5px;
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 0.25rem;
}

.pulse-dot {
  width: 12px;
  height: 12px;
  border-radius: 50%;
  animation: pulseGlow 2s infinite alternate;
}

.branding p {
  color: #94a3b8;
  font-size: 0.95rem;
  margin: 0;
}

.divider {
  border: 0;
  height: 1px;
  background: linear-gradient(
    to right,
    rgba(255, 255, 255, 0),
    rgba(255, 255, 255, 0.15),
    rgba(255, 255, 255, 0)
  );
  margin: 2rem 0;
}

/* Audio Trigger Banner Box */
.activation-banner {
  background: linear-gradient(135deg, rgba(66, 184, 131, 0.15) 0%, rgba(33, 53, 71, 0.3) 100%);
  border: 1px dashed #42b883;
  padding: 1.5rem;
  border-radius: 16px;
  text-align: center;
  cursor: pointer;
  margin-bottom: 1.5rem;
  transition:
    transform 0.2s,
    background 0.2s;
}

.activation-banner:hover {
  transform: scale(1.02);
  background: linear-gradient(135deg, rgba(66, 184, 131, 0.25) 0%, rgba(33, 53, 71, 0.4) 100%);
}

.icon-pulse {
  font-size: 2rem;
  animation: float 1.5s ease-in-out infinite alternate;
}

.activation-banner h3 {
  margin: 0.5rem 0 0.25rem 0;
  color: #fff;
}

/* Controls Matrix */
.controls-matrix {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
  margin-bottom: 2rem;
}

.control-card label {
  display: block;
  font-size: 0.8rem;
  text-transform: uppercase;
  letter-spacing: 1.5px;
  color: #64748b;
  margin-bottom: 0.75rem;
  font-weight: 700;
}

.selector-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 0.75rem;
}

.selector-grid button {
  background: rgba(255, 255, 255, 0.03);
  border: 1px solid rgba(255, 255, 255, 0.08);
  color: #94a3b8;
  padding: 0.75rem;
  border-radius: 10px;
  cursor: pointer;
  font-weight: 600;
  transition: all 0.2s ease;
}

.selector-grid button:hover {
  background: rgba(255, 255, 255, 0.08);
  color: white;
}

.selector-grid button.active {
  background: rgba(255, 255, 255, 0.05);
  font-weight: 700;
  box-shadow: inset 0 0 10px rgba(255, 255, 255, 0.05);
}

/* Custom Telemetry Sliders */
.neon-slider {
  -webkit-appearance: none;
  appearance: none;
  width: 100%;
  height: 6px;
  background: #1e293b;
  border-radius: 10px;
  outline: none;
}

.neon-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: var(--thumb-color, #42b883);
  cursor: pointer;
  box-shadow: 0 0 10px var(--thumb-color, #42b883);
  transition: transform 0.1s;
}

.neon-slider::-webkit-slider-thumb:active {
  transform: scale(1.3);
}

/* Telemetry Dashboard Data Deck */
.telemetry-deck {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 1rem;
  background: rgba(0, 0, 0, 0.25);
  padding: 1.25rem;
  border-radius: 14px;
  border: 1px solid rgba(255, 255, 255, 0.03);
}

.metric {
  display: flex;
  flex-direction: column;
}

.metric .label {
  font-size: 0.75rem;
  color: #475569;
  text-transform: uppercase;
  font-weight: bold;
}

.metric .val {
  font-size: 1.1rem;
  font-weight: 700;
  margin-top: 0.2rem;
  font-family: monospace;
}

.text-dim {
  color: #64748b;
}

/* Instruction Pad Footer */
.instruction-pad {
  margin-top: 1.5rem;
  padding: 1rem;
  background: rgba(255, 255, 255, 0.02);
  border-left: 3px solid;
  border-radius: 4px 12px 12px 4px;
  font-size: 0.85rem;
  line-height: 1.5;
  color: #94a3b8;
}

/* Particle Particles Styles Matrix */
.quantum-particle {
  position: absolute;
  border-radius: 50%;
  pointer-events: none;
  transform: translate(-50%, -50%);
  animation: particleFade 1.2s cubic-bezier(0.1, 0.8, 0.3, 1) forwards;
}

/* Animations Matrix Framework */
@keyframes particleFade {
  0% {
    transform: translate(-50%, -50%) scale(0.4);
    opacity: 1;
  }
  100% {
    transform: translate(-50%, -50%) scale(2.5);
    opacity: 0;
  }
}

@keyframes pulseGlow {
  from {
    transform: scale(0.9);
    opacity: 0.6;
    box-shadow: 0 0 4px var(--thumb-color);
  }
  to {
    transform: scale(1.1);
    opacity: 1;
    box-shadow: 0 0 16px var(--thumb-color);
  }
}

@keyframes float {
  from {
    transform: translateY(0px);
  }
  to {
    transform: translateY(-6px);
  }
}
</style>
