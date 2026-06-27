<script setup lang="ts">
import { computed } from 'vue'
import winters from '../data/winters.json'

const props = withDefaults(
  defineProps<{
    activeBeat?: number
  }>(),
  {
    activeBeat: 1,
  },
)

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

const spreadPoints = computed(() => getTrailingSpread())

const spreadAreaPath = computed(() => {
  if (spreadPoints.value.length < 2) {
    return ''
  }

  const upperPath = spreadPoints.value
    .map((point, index) => `${index === 0 ? 'M' : 'L'} ${x(point.year)} ${y(point.max)}`)
    .join(' ')

  const lowerPath = [...spreadPoints.value]
    .reverse()
    .map((point) => `L ${x(point.year)} ${y(point.min)}`)
    .join(' ')

  return `${upperPath} ${lowerPath} Z`
})

const spreadUpperEdgePath = computed(() => {
  if (spreadPoints.value.length < 2) {
    return ''
  }

  return spreadPoints.value
    .map((point, index) => `${index === 0 ? 'M' : 'L'} ${x(point.year)} ${y(point.max)}`)
    .join(' ')
})

const warmest = data.find((row) => row.year === 2024)
const coldest = data.find((row) => row.year === 2014)

const rampOpacity = computed(() => {
  if (props.activeBeat === 1) {
    return 1
  }
  if (props.activeBeat === 2) {
    return 0.15
  }
  return 0
})

const stepOpacity = computed(() => {
  if (props.activeBeat === 2) {
    return 1
  }
  if (props.activeBeat >= 3) {
    return 0.15
  }
  return 0
})

const bandOpacity = computed(() => (props.activeBeat >= 3 ? 1 : 0))
</script>

<template>
  <svg
    class="winter-chart"
    viewBox="0 0 960 600"
    width="100%"
    height="100%"
    preserveAspectRatio="xMidYMid meet"
    role="img"
    aria-label="Scatter plot of Minneapolis winter average overnight lows from 1951 to 2025"
  >
    <rect x="0" y="0" width="960" height="600" fill="var(--night)" />

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

    <line
      :x1="padding.left"
      :y1="y(meanAvgLow)"
      :x2="viewBoxWidth - padding.right"
      :y2="y(meanAvgLow)"
      class="winter-chart__mean-line"
    />

    <text :x="viewBoxWidth - padding.right" :y="y(meanAvgLow) - 8" class="winter-chart__label winter-chart__label--right">
      75-yr avg
    </text>

    <g class="winter-chart__base-dots">
      <circle
        v-for="row in data"
        :key="row.year"
        :cx="x(row.year)"
        :cy="y(row.avgLow)"
        r="3.5"
        class="winter-chart__dot"
      />
    </g>

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
        :y="y(regression.startTemp + (regression.endTemp - regression.startTemp) * 0.45) - 12"
        class="winter-chart__annotation"
      >
        the steady ramp you'd expect
      </text>
    </g>

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

    <g class="winter-chart__layer" :style="{ opacity: bandOpacity }">
      <path :d="spreadAreaPath" class="winter-chart__band-area" />
      <path :d="spreadUpperEdgePath" class="winter-chart__band-edge" />

      <g v-if="warmest" class="winter-chart__record-group">
        <circle :cx="x(warmest.year)" :cy="y(warmest.avgLow)" r="5" class="winter-chart__record-dot--warm" />
        <text :x="x(warmest.year) - 12" :y="y(warmest.avgLow) - 12" class="winter-chart__annotation winter-chart__annotation--warm">
          warmest in 75 yrs
        </text>
      </g>

      <g v-if="coldest" class="winter-chart__record-group">
        <circle :cx="x(coldest.year)" :cy="y(coldest.avgLow)" r="5" class="winter-chart__record-dot--cold" />
        <text :x="x(coldest.year) + 12" :y="y(coldest.avgLow) + 6" class="winter-chart__annotation winter-chart__annotation--cold">
          coldest in 75 yrs
        </text>
      </g>
    </g>

    <g>
      <text
        v-for="tick in xTicks"
        :key="`x-label-${tick}`"
        :x="x(tick)"
        :y="viewBoxHeight - 28"
        text-anchor="middle"
        class="winter-chart__label"
      >
        {{ tick }}
      </text>

      <text
        v-for="tick in yTicks"
        :key="`y-label-${tick}`"
        :x="padding.left - 12"
        :y="y(tick) + 4"
        text-anchor="end"
        class="winter-chart__label"
      >
        {{ tick }}
      </text>
    </g>
  </svg>
</template>