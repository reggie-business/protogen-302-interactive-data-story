<script setup lang="ts">
import { computed, onBeforeUnmount, ref, watch } from 'vue'
import winters from '../data/winters.json'

const props = withDefaults(
  defineProps<{
    activeBeat?: number
  }>(),
  {
    activeBeat: 1,
  },
)

const isPayoffBeat = computed(() => props.activeBeat === 4)

type WinterRow = {
  year: number
  avgLow: number
  avgHigh: number
  subzero: number
}

const data = winters as WinterRow[]

const viewBoxWidth = 960
const viewBoxHeight = 600
const padding = { top: 56, right: 56, bottom: 56, left: 56 }

const plotWidth = viewBoxWidth - padding.left - padding.right
const plotHeight = viewBoxHeight - padding.top - padding.bottom
const xMin = 1951
const xMax = 2025
const yMin = -2
const yMax = 24

const x = (year: number) => padding.left + ((year - xMin) / (xMax - xMin)) * plotWidth
const y = (temperature: number) =>
  padding.top + ((yMax - temperature) / (yMax - yMin)) * plotHeight

const meanAvgLow = data.reduce((sum, row) => sum + row.avgLow, 0) / data.length

const xTicks = [1955, 1970, 1985, 2000, 2015]
const yTicks = [0, 5, 10, 15, 20]

const decadeSteps = [
  { start: 1951, end: 1959, mean: 8.1 },
  { start: 1960, end: 1969, mean: 7.7 },
  { start: 1970, end: 1979, mean: 7.0 },
  { start: 1980, end: 1989, mean: 10.0 },
  { start: 1990, end: 1999, mean: 11.5 },
  { start: 2000, end: 2009, mean: 11.4 },
  { start: 2010, end: 2019, mean: 10.1 },
  { start: 2020, end: 2025, mean: 10.4 },
]

const regression = computed(() => {
  const n = data.length
  const sumX = data.reduce((sum, row) => sum + row.year, 0)
  const sumY = data.reduce((sum, row) => sum + row.avgLow, 0)
  const sumXY = data.reduce((sum, row) => sum + row.year * row.avgLow, 0)
  const sumXX = data.reduce((sum, row) => sum + row.year * row.year, 0)
  const denominator = n * sumXX - sumX * sumX
  const slope = denominator === 0 ? 0 : (n * sumXY - sumX * sumY) / denominator
  const intercept = n === 0 ? 0 : (sumY - slope * sumX) / n
  const startTemp = intercept + slope * xMin
  const endTemp = intercept + slope * xMax

  return {
    startTemp,
    endTemp,
  }
})

type SpreadPoint = {
  year: number
  min: number
  max: number
}

const getTrailingSpread = (upToYear = 2025): SpreadPoint[] => {
  const included = data.filter((row) => row.year <= upToYear)

  return included.map((row, index) => {
    const windowStart = Math.max(0, index - 14)
    const windowSlice = included.slice(windowStart, index + 1)
    const values = windowSlice.map((item) => item.avgLow)

    return {
      year: row.year,
      min: Math.min(...values),
      max: Math.max(...values),
    }
  })
}

// ─── Path helpers ────────────────────────────────────────────────
function buildAreaPath(points: SpreadPoint[]): string {
  if (points.length < 2) return ''
  const upper = points
    .map((p, i) => `${i === 0 ? 'M' : 'L'} ${x(p.year)} ${y(p.max)}`)
    .join(' ')
  const lower = [...points]
    .reverse()
    .map((p) => `L ${x(p.year)} ${y(p.min)}`)
    .join(' ')
  return `${upper} ${lower} Z`
}

function buildUpperEdgePath(points: SpreadPoint[]): string {
  if (points.length < 2) return ''
  return points
    .map((p, i) => `${i === 0 ? 'M' : 'L'} ${x(p.year)} ${y(p.max)}`)
    .join(' ')
}

// Static full-2025 band (beats 3)
const spreadPoints = computed(() => getTrailingSpread())
const spreadAreaPath = computed(() => buildAreaPath(spreadPoints.value))
const spreadUpperEdgePath = computed(() => buildUpperEdgePath(spreadPoints.value))

const warmest = data.find((row) => row.year === 2024)
const coldest = data.find((row) => row.year === 2014)

// ─── Scrubber state (beat 4) ──────────────────────────────────────
const scrubYear = ref(2025)
const isPlaying = ref(false)
let rafId: number | null = null
let playStartTime: number | null = null
const PLAY_DURATION_MS = 8000

watch(
  () => props.activeBeat,
  (beat) => {
    if (beat !== 4) {
      stopPlay()
      scrubYear.value = 2025
    }
  },
)

function stopPlay() {
  isPlaying.value = false
  if (rafId !== null) {
    cancelAnimationFrame(rafId)
    rafId = null
  }
}

function startPlay() {
  stopPlay()
  if (scrubYear.value >= 2025) scrubYear.value = 1951
  isPlaying.value = true
  playStartTime = null
  const startYear = scrubYear.value

  function tick(timestamp: number) {
    if (!isPlaying.value) return
    if (playStartTime === null) playStartTime = timestamp
    const elapsed = timestamp - playStartTime
    const fraction = Math.min(elapsed / PLAY_DURATION_MS, 1)
    scrubYear.value = Math.round(startYear + fraction * (2025 - startYear))
    if (fraction >= 1) {
      scrubYear.value = 2025
      isPlaying.value = false
      rafId = null
      return
    }
    rafId = requestAnimationFrame(tick)
  }

  rafId = requestAnimationFrame(tick)
}

function togglePlay() {
  if (isPlaying.value) {
    stopPlay()
  } else {
    startPlay()
  }
}

onBeforeUnmount(() => stopPlay())

// Scrub-driven spread band
const scrubSpreadPoints = computed(() => getTrailingSpread(scrubYear.value))
const scrubSpreadAreaPath = computed(() => buildAreaPath(scrubSpreadPoints.value))
const scrubSpreadUpperEdgePath = computed(() => buildUpperEdgePath(scrubSpreadPoints.value))

const scrubRow = computed(
  () => data.find((r) => r.year === scrubYear.value) ?? data[data.length - 1],
)

const scrubSwing = computed(() => {
  const pts = scrubSpreadPoints.value
  if (pts.length === 0) return 0
  const last = pts[pts.length - 1]
  return +(last.max - last.min).toFixed(1)
})

const scrubTag = computed(() => {
  if (scrubYear.value === 2024) return 'warmest in 75 yrs'
  if (scrubYear.value === 2014) return 'coldest in 75 yrs'
  return null
})

// Callout card geometry
const calloutWidth = 232
const calloutPad = 10
const calloutLineH = 20
const calloutHeight = computed(() => (scrubTag.value ? 3 : 2) * calloutLineH + calloutPad * 2)

const calloutX = computed(() => {
  const cx = x(scrubYear.value)
  return cx + calloutWidth + 24 > viewBoxWidth - padding.right ? cx - calloutWidth - 10 : cx + 16
})

const calloutY = computed(() => {
  const cy = y(scrubRow.value.avgLow)
  const h = calloutHeight.value
  return Math.max(padding.top + 4, Math.min(cy - h / 2, viewBoxHeight - padding.bottom - h - 4))
})

function dotOpacity(year: number): number {
  if (props.activeBeat < 4) return 0.4
  return year <= scrubYear.value ? 0.55 : 0.06
}

// ─── Beat layer opacities ─────────────────────────────────────────
const rampOpacity = computed(() => {
  if (props.activeBeat === 1) return 1
  if (props.activeBeat === 2) return 0.15
  return 0
})

const stepOpacity = computed(() => {
  if (props.activeBeat === 2) return 1
  if (props.activeBeat >= 3) return 0.15
  return 0
})

const bandOpacity = computed(() => (props.activeBeat === 3 ? 1 : 0))
const scrubBandOpacity = computed(() => (props.activeBeat === 4 ? 1 : 0))

</script>

<template>
  <div class="winter-chart-wrap">
    <div class="winter-chart-title-bar">
      <img src="/mn-seal.png" alt="Minnesota seal" class="winter-chart-seal" />
    </div>

    <svg
      class="winter-chart"
      viewBox="0 0 960 600"
      width="100%"
      preserveAspectRatio="xMidYMid meet"
      role="img"
      aria-label="Scatter plot of Minneapolis winter average overnight lows from 1951 to 2025"
    >
      <rect x="0" y="0" width="960" height="600" fill="var(--night)" />

      <!-- Grid lines -->
      <g>
        <line
          v-for="tick in xTicks"
          :key="`x-grid-${tick}`"
          :x1="x(tick)"
          :y1="padding.top"
          :x2="x(tick)"
          :y2="viewBoxHeight - padding.bottom"
          class="winter-chart__grid-line"
        />
        <line
          v-for="tick in yTicks"
          :key="`y-grid-${tick}`"
          :x1="padding.left"
          :y1="y(tick)"
          :x2="viewBoxWidth - padding.right"
          :y2="y(tick)"
          class="winter-chart__grid-line"
        />
      </g>

      <!-- 75-yr mean reference -->
      <line
        :x1="padding.left"
        :y1="y(meanAvgLow)"
        :x2="viewBoxWidth - padding.right"
        :y2="y(meanAvgLow)"
        class="winter-chart__mean-line"
      />
      <text
        v-if="!isPayoffBeat"
        :x="padding.left + 6"
        :y="y(meanAvgLow) - 8"
        class="winter-chart__label"
      >75-yr avg</text>

      <!-- Base dots: opacity driven per dot when in beat 4 -->
      <g>
        <circle
          v-for="row in data"
          :key="row.year"
          :cx="x(row.year)"
          :cy="y(row.avgLow)"
          r="3.9"
          :style="{ opacity: dotOpacity(row.year) }"
          class="winter-chart__dot"
        />
      </g>

      <!-- Beat 1: regression ramp -->
      <g class="winter-chart__layer" :style="{ opacity: rampOpacity }">
        <line
          :x1="x(xMin)"
          :y1="y(regression.startTemp)"
          :x2="x(xMax)"
          :y2="y(regression.endTemp)"
          class="winter-chart__ramp-line"
        />
        <text
          :x="x(1974)"
          :y="y(12.5)"
          class="winter-chart__annotation"
        >the steady ramp you'd expect</text>
      </g>

      <!-- Beat 2: decade step line -->
      <g class="winter-chart__layer" :style="{ opacity: stepOpacity }">
        <line
          v-for="segment in decadeSteps"
          :key="`step-${segment.start}`"
          :x1="x(segment.start)"
          :y1="y(segment.mean)"
          :x2="x(segment.end)"
          :y2="y(segment.mean)"
          class="winter-chart__step-line"
        />
        <line
          v-for="segment in decadeSteps.slice(0, -1)"
          :key="`step-connector-${segment.start}`"
          :x1="x(segment.end)"
          :y1="y(segment.mean)"
          :x2="x(segment.end)"
          :y2="y(decadeSteps[decadeSteps.indexOf(segment) + 1].mean)"
          class="winter-chart__step-line"
        />
      </g>

      <!-- Beat 3: static full band (upToYear = 2025) -->
      <g class="winter-chart__layer" :style="{ opacity: bandOpacity }">
        <path :d="spreadAreaPath" class="winter-chart__band-area" />
        <path :d="spreadUpperEdgePath" class="winter-chart__band-edge" />
        <g v-if="warmest" class="winter-chart__record-group">
          <circle :cx="x(warmest.year)" :cy="y(warmest.avgLow)" r="5.5" class="winter-chart__record-dot--warm" />
          <text
            :x="x(warmest.year) - 14"
            :y="y(warmest.avgLow) - 12"
            text-anchor="end"
            class="winter-chart__annotation winter-chart__annotation--warm"
          >warmest in 75 yrs</text>
        </g>
        <g v-if="coldest" class="winter-chart__record-group">
          <circle :cx="x(coldest.year)" :cy="y(coldest.avgLow)" r="5" class="winter-chart__record-dot--cold" />
          <text
            :x="x(coldest.year) + 12"
            :y="y(coldest.avgLow) + 6"
            class="winter-chart__annotation winter-chart__annotation--cold"
          >coldest in 75 yrs</text>
        </g>
      </g>

      <!-- Beat 4: scrubYear-driven band + playhead + callout -->
      <g class="winter-chart__layer" :style="{ opacity: scrubBandOpacity }">
        <path :d="scrubSpreadAreaPath" class="winter-chart__band-area" />
        <path :d="scrubSpreadUpperEdgePath" class="winter-chart__band-edge" />

        <g v-if="warmest && scrubYear >= 2024" class="winter-chart__record-group">
          <circle :cx="x(warmest.year)" :cy="y(warmest.avgLow)" r="5.5" class="winter-chart__record-dot--warm" />
          <text
            :x="x(warmest.year) - 14"
            :y="y(warmest.avgLow) - 12"
            text-anchor="end"
            class="winter-chart__annotation winter-chart__annotation--warm"
          >warmest in 75 yrs</text>
        </g>
        <g v-if="coldest && scrubYear >= 2014" class="winter-chart__record-group">
          <circle :cx="x(coldest.year)" :cy="y(coldest.avgLow)" r="5" class="winter-chart__record-dot--cold" />
          <text
            :x="x(coldest.year) + 12"
            :y="y(coldest.avgLow) + 6"
            class="winter-chart__annotation winter-chart__annotation--cold"
          >coldest in 75 yrs</text>
        </g>

        <!-- Playhead -->
        <line
          :x1="x(scrubYear)"
          :y1="padding.top"
          :x2="x(scrubYear)"
          :y2="viewBoxHeight - padding.bottom"
          class="winter-chart__playhead"
        />
        <circle
          :cx="x(scrubYear)"
          :cy="y(scrubRow.avgLow)"
          r="7.8"
          class="winter-chart__playhead-dot"
        />

        <!-- Callout card -->
        <rect
          :x="calloutX - calloutPad"
          :y="calloutY - calloutPad"
          :width="calloutWidth"
          :height="calloutHeight"
          rx="4"
          class="winter-chart__callout-bg"
        />
        <text :x="calloutX" :y="calloutY + calloutLineH" class="winter-chart__callout-text">
          {{ scrubRow.year }} · avg low {{ scrubRow.avgLow }}°F
        </text>
        <text :x="calloutX" :y="calloutY + calloutLineH * 2" class="winter-chart__callout-text">
          {{ scrubRow.subzero }} nights below 0°F
        </text>
        <text
          v-if="scrubTag"
          :x="calloutX"
          :y="calloutY + calloutLineH * 3"
          class="winter-chart__callout-tag"
        >{{ scrubTag }}</text>
      </g>

      <!-- Axis labels (always on top) -->
      <g>
        <text
          v-for="tick in xTicks"
          :key="`x-label-${tick}`"
          :x="x(tick)"
          :y="viewBoxHeight - 28"
          text-anchor="middle"
          class="winter-chart__label"
        >{{ tick }}</text>
        <text
          v-for="tick in yTicks"
          :key="`y-label-${tick}`"
          :x="padding.left - 12"
          :y="y(tick) + 4"
          text-anchor="end"
          class="winter-chart__label"
        >{{ tick }}</text>
      </g>
    </svg>

    <!-- Scrubber UI: only visible in beat 4 -->
    <div v-if="isPayoffBeat" class="scrubber">
      <div class="scrubber__row">
        <button
          class="scrubber__play-btn"
          :aria-label="isPlaying ? 'Pause' : 'Play'"
          @click="togglePlay"
        >
          {{ isPlaying ? '⏸' : '▶' }}
        </button>
        <input
          type="range"
          class="scrubber__track"
          min="1951"
          max="2025"
          step="1"
          v-model.number="scrubYear"
          aria-label="Scrub through winters"
          @pointerdown="stopPlay"
        />
        <span class="scrubber__year">{{ scrubYear }}</span>
      </div>
      <p class="scrubber__readout">
        Swing of the last 15 winters:
        <strong>{{ scrubSwing }}°F</strong>
      </p>
      <p class="scrubber__helper">Drag slider for details.</p>
    </div>
  </div>
</template>