<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref, type ComponentPublicInstance } from 'vue'
import WinterChart from './components/WinterChart.vue'

type Beat = {
  id: number
  title: string
  copy: string
}

const beats: Beat[] = [
  {
    id: 1,
    title: 'The snow fort',
    copy: "Winter is supposed to keep us cold. It's part of our constitution, and our identity.",
  },
  {
    id: 2,
    title: 'The step',
    copy: 'Warming jumped in the 1980s and early 90s, then flattened out.',
  },
  {
    id: 3,
    title: 'The spread',
    copy: 'Seasonal variance widened. Nowadays, winters swing from deep freeze to unexpected thaw (the perfect recipe for ubiquitous potholes).',
  },
  {
    id: 4,
    title: 'Heads or tails?',
    copy: 'Winter now behaves like a coin flip, changing how the Cities plan and slowly defrosting how we view our home.',
  },
]

const activeBeat = ref(1)
const beatElements = ref<(HTMLElement | null)[]>([])
const hasUserScrolled = ref(false)
let observer: IntersectionObserver | null = null

const onScroll = () => {
  if (!hasUserScrolled.value && window.scrollY > 16) {
    hasUserScrolled.value = true
  }
}

const registerBeat = (index: number) => (element: Element | ComponentPublicInstance | null) => {
  beatElements.value[index] = element instanceof HTMLElement ? element : null
}

onMounted(() => {
  window.addEventListener('scroll', onScroll, { passive: true })

  observer = new IntersectionObserver(
    (entries) => {
      const activeEntry = entries.find((entry) => entry.isIntersecting)

      if (!activeEntry) {
        return
      }

      const target = activeEntry.target as HTMLElement
      const index = Number(target.dataset.beatIndex)

      if (!Number.isNaN(index)) {
        activeBeat.value = index + 1
      }
    },
    {
      root: null,
      threshold: 0,
      rootMargin: '-50% 0px -50% 0px',
    },
  )

  beatElements.value.forEach((element) => {
    if (element) {
      observer?.observe(element)
    }
  })
})

onBeforeUnmount(() => {
  window.removeEventListener('scroll', onScroll)
  observer?.disconnect()
  observer = null
})
</script>

<template>
  <main class="story-shell">
    <section class="story-scene" aria-label="Story scene">
      <div class="chart-stage" aria-label="Sticky chart stage">
        <div class="stage-frame">
          <WinterChart :active-beat="activeBeat" />
        </div>
      </div>

      <div
        class="scroll-hint"
        :class="{ 'is-hidden': hasUserScrolled }"
        aria-hidden="true"
      >
        <span>Scroll</span>
        <span class="scroll-hint__chevron">⌄</span>
      </div>

      <section class="beats" aria-label="Story beats">
        <article
          v-for="(beat, index) in beats"
          :key="beat.id"
          :ref="registerBeat(index)"
          :data-beat-index="index"
          class="beat"
        >
          <h2>{{ beat.title }}</h2>
          <p>{{ beat.copy }}</p>
        </article>
      </section>
    </section>

    <footer class="footer">
      <p>
        Temperature data:
        <a href="https://open-meteo.com/" target="_blank" rel="noopener">Open-Meteo Historical Weather API</a>
        (ERA5), CC BY 4.0.
      </p>
      <p>
        Corroborating context: Minnesota DNR
        <a href="https://www.dnr.state.mn.us/" target="_blank" rel="noopener">State Climatology Office</a>.
      </p>
    </footer>
  </main>
</template>
