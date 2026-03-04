<script setup lang="ts">
import ComparePanel from './components/ComparePanel.vue'
import DimensionsPanel from './components/DimensionsPanel.vue'
import ImageEditorPanel from './components/ImageEditorPanel.vue'
import TopBar from './components/TopBar.vue'
</script>

<template>
  <div class="page">
    <TopBar />

    <main class="main-grid">
      <ImageEditorPanel />
      <ComparePanel />
    </main>

    <DimensionsPanel />
  </div>
</template>
