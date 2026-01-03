<template>
  <div class="app-container">
    <div class="visualization-wrapper">
      <TimelineVisualization ref="visualizationRef" @data-updated="handleDataUpdated" />
    </div>
    <DataTable :data="tableData" :events="eventData" @row-click="navigateToPoint" />
  </div>
</template>

<script setup>
import { ref } from 'vue'
import TimelineVisualization from './components/TimelineVisualization.vue'
import DataTable from './components/DataTable.vue'

const visualizationRef = ref(null)
const tableData = ref([])
const eventData = ref([])

// Handle data from visualization component
function handleDataUpdated(payload) {
  tableData.value = payload.data || payload
  eventData.value = payload.events || []
}

// Navigate to point when row is clicked
function navigateToPoint(item) {
  if (visualizationRef.value && visualizationRef.value.animateCameraToPoint) {
    visualizationRef.value.animateCameraToPoint(item)
  }
}
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html, body {
  margin: 0;
  padding: 0;
  overflow-y: auto;
  overflow-x: hidden;
  font-family: Arial, sans-serif;
}

#app,
.app-container {
  width: 100%;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.visualization-wrapper {
/*  padding: 0 2rem; */
  background: #1a1a1a;
  overflow: hidden;
}
</style>