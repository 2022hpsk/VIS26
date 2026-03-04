<script setup lang="ts">
interface WeightItem {
  label: string
  value: number
  tone?: 'pink' | 'mint' | 'blue'
}

interface Props {
  title?: string
  promptPlaceholder?: string
  weights?: WeightItem[]
  compareImage?: string
  referenceImage?: string
  galleryImages?: string[]
}

const props = withDefaults(defineProps<Props>(), {
  title: 'Compare',
  promptPlaceholder: 'Describe the style, palette, and mood',
  weights: () => [
    { label: 'Texture', value: 60, tone: 'pink' },
    { label: 'Structure', value: 40, tone: 'mint' },
    { label: 'Color', value: 60, tone: 'blue' }
  ],
  compareImage:
    'https://images.unsplash.com/photo-1500530855697-b586d89ba3ee?auto=format&fit=crop&w=900&q=80',
  referenceImage:
    'https://images.unsplash.com/photo-1500530855697-b586d89ba3ee?auto=format&fit=crop&w=800&q=80',
  galleryImages: () => [
    'https://images.unsplash.com/photo-1474511320723-9a56873867b5?auto=format&fit=crop&w=300&q=80',
    'https://images.unsplash.com/photo-1452570053594-1b985d6ea890?auto=format&fit=crop&w=300&q=80',
    'https://images.unsplash.com/photo-1465156799763-2c087c332922?auto=format&fit=crop&w=300&q=80',
    'https://images.unsplash.com/photo-1470104240373-bc1812eddc9f?auto=format&fit=crop&w=300&q=80',
    'https://images.unsplash.com/photo-1480044965905-02098d419e96?auto=format&fit=crop&w=300&q=80',
    'https://images.unsplash.com/photo-1444464666168-49d633b86797?auto=format&fit=crop&w=300&q=80'
  ]
})
</script>

<template>
  <section class="panel panel-compare">
    <div class="compare-grid">
      <div class="compare-column">
        <div class="prompt-card">
          <div class="prompt-input">
            <span class="prompt-icon">Q</span>
            <slot name="prompt">
              <input
                type="text"
                :placeholder="props.promptPlaceholder"
                aria-label="prompt"
              />
            </slot>
          </div>
          <div class="weights">
            <div class="weights-title">Field weights</div>
            <slot name="fieldweights">
              <div
                v-for="weight in props.weights"
                :key="weight.label"
                class="weight-row"
              >
                <span>{{ weight.label }}</span>
                <div class="weight-track">
                  <span
                    class="weight-fill"
                    :class="{
                      'is-pink': weight.tone === 'pink',
                      'is-mint': weight.tone === 'mint',
                      'is-blue': weight.tone === 'blue'
                    }"
                    :style="{ width: `${weight.value}%` }"
                  ></span>
                </div>
                <span class="weight-value">{{ weight.value }}%</span>
              </div>
            </slot>
          </div>
        </div>
      </div>

      <div class="compare-column">
        <slot name="compare">
          <div class="orbit">
            <div class="orbit-core">
              <img :src="props.compareImage" alt="center" />
            </div>
            <div class="orbit-ring ring-1"></div>
            <div class="orbit-ring ring-2"></div>
            <div class="orbit-ring ring-3"></div>
            <div class="orbit-node n1"></div>
            <div class="orbit-node n2"></div>
            <div class="orbit-node n3"></div>
            <div class="orbit-node n4"></div>
            <div class="orbit-node n5"></div>
            <div class="orbit-node n6"></div>
            <div class="orbit-node n7"></div>
            <div class="orbit-node n8"></div>
          </div>
        </slot>
      </div>

      <div class="compare-column">
        <div class="panel-block">
          <div class="panel-subtitle">Image</div>
          <slot name="image">
            <div class="hero-image">
              <img :src="props.referenceImage" alt="reference" />
            </div>
          </slot>
        </div>
        <div class="panel-block">
          <div class="panel-subtitle">Gallery</div>
          <slot name="gallery">
            <div class="gallery-grid">
              <img
                v-for="(image, index) in props.galleryImages"
                :key="`${image}-${index}`"
                :src="image"
                :alt="`thumb ${index + 1}`"
              />
            </div>
          </slot>
        </div>
      </div>
    </div>
  </section>
</template>
