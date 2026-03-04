<script setup lang="ts">
interface Props {
  title?: string
  actionLabel?: string
}

const props = withDefaults(defineProps<Props>(), {
  title: 'P2P-AVA',
  actionLabel: 'Peacock Pheasant'
})
</script>

<template>
  <header class="topbar">
    <div class="brand">
      <slot name="title">{{ props.title }}</slot>
    </div>
    <slot name="action">
      <button class="pill-btn" type="button">{{ props.actionLabel }}</button>
    </slot>
  </header>
</template>
