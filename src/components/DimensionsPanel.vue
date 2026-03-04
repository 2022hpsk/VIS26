<script setup lang="ts">
interface Props {
  title?: string
  labels?: string[]
}

const props = withDefaults(defineProps<Props>(), {
  title: 'Dimensions',
  labels: () => ['sparrow', 'golden pheasant', 'red-crowned crane', 'sparrow', 'red-crowned crane']
})
</script>

<template>
  <section class="panel panel-bottom">
    <div class="panel-title">{{ props.title }}</div>
    <div class="dimension-grid">
      <div class="dimension-legend">
        <slot name="legend">
          <div class="legend-title">History</div>
          <div class="legend-item">
            <span class="legend-dot dot-tang"></span>
            Tang
          </div>
          <div class="legend-item">
            <span class="legend-dot dot-song"></span>
            Song
          </div>
          <div class="legend-item">
            <span class="legend-dot dot-yuan"></span>
            Yuan
          </div>
          <div class="legend-item">
            <span class="legend-dot dot-ming"></span>
            Ming
          </div>
        </slot>
      </div>
      <div class="ridgeline">
        <slot name="chart">
          <div class="ridge" style="--ridge-color: var(--accent-pink)"></div>
          <div class="ridge" style="--ridge-color: var(--accent-mint)"></div>
          <div class="ridge" style="--ridge-color: var(--accent-blue)"></div>
          <div class="ridge" style="--ridge-color: var(--accent-orange)"></div>
          <div class="ridge-labels">
            <span v-for="label in props.labels" :key="label">{{ label }}</span>
          </div>
        </slot>
      </div>
    </div>
  </section>
</template>
