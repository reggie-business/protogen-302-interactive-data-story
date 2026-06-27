<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref, type ComponentPublicInstance } from 'vue'

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

const activeBeatCard = computed(() => beats[activeBeat.value - 1] ?? beats[0])

const registerBeat = (index: number) => (element: Element | ComponentPublicInstance | null) => {
  beatElements.value[index] = element instanceof HTMLElement ? element : null
}

onMounted(() => {
  observer = new IntersectionObserver(
    (entries) => {
      const visibleEntries = entries.filter((entry) => entry.isIntersecting)

      if (!visibleEntries.length) {
        return
      }

      visibleEntries.sort((left, right) => right.intersectionRatio - left.intersectionRatio)

      const target = visibleEntries[0].target as HTMLElement
      const index = Number(target.dataset.beatIndex)

      if (!Number.isNaN(index)) {
        activeBeat.value = index + 1
      }
    },
    {
      root: null,
      threshold: [0, 0.25, 0.5, 0.75],
      rootMargin: '-42% 0px -42% 0px',
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
    <section class="story-stage" aria-label="Sticky chart stage">
      <div class="stage-frame">
        <p class="stage-kicker">Sticky chart stage</p>
        <div class="stage-placeholder" aria-live="polite">
          <p class="stage-placeholder__label">Active beat</p>
          <p class="stage-placeholder__value">{{ activeBeat }}</p>
          <h2 class="stage-placeholder__title">{{ activeBeatCard.title }}</h2>
          <p class="stage-placeholder__copy">{{ activeBeatCard.copy }}</p>
        </div>
      </div>
    </section>

    <section class="story-beats" aria-label="Story beats">
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
  </main>
</template>
