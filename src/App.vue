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
    title: 'The fortress',
    copy: 'Winter was supposed to keep ramping colder. Minneapolis lived inside that belief.',
  },
  {
    id: 2,
    title: 'The step',
    copy: 'The warming jumped in the 1980s and early 1990s, then flattened out.',
  },
  {
    id: 3,
    title: 'The swing',
    copy: 'The spread widened. Recent winters swing from deep freeze to thaw.',
  },
  {
    id: 4,
    title: 'The payoff',
    copy: 'The winter now behaves like a coin flip, and that changes how the city plans.',
  },
]

const activeBeat = ref(1)
const beatElements = ref<(HTMLElement | null)[]>([])
let observer: IntersectionObserver | null = null

const registerBeat = (index: number) => (element: Element | ComponentPublicInstance | null) => {
  beatElements.value[index] = element instanceof HTMLElement ? element : null
}

onMounted(() => {
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

      <section class="beats" aria-label="Story beats">
        <article
          v-for="(beat, index) in beats"
          :key="beat.id"
          :ref="registerBeat(index)"
          :data-beat-index="index"
          class="beat"
        >
          <p class="beat-kicker">Beat {{ beat.id }}</p>
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
