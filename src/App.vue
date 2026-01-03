<template>
  <div id="app">
    <TimelineVisualization ref="visualizationRef" @data-updated="handleDataUpdated" />
    <div class="data-table-section">
      <h2>Data Details</h2>
      <table class="data-table">
        <thead>
          <tr>
            <th>Date</th>
            <th>Label</th>
            <th>Count</th>
            <th>Position (X, Y, Z)</th>
            <th>Color</th>
          </tr>
        </thead>
        <tbody>
          <tr
            v-for="(item, index) in tableData"
            :key="index"
            @click="navigateToPoint(item)"
            class="clickable-row"
          >
            <td>{{ formatDate(item.timestamp) }}</td>
            <td>{{ item.label }}</td>
            <td>{{ item.count }}</td>
            <td>({{ item.x }}, {{ item.y }}, {{ item.z }})</td>
            <td>
              <span class="color-badge" :style="{ background: item.color }"></span>
              {{ item.color }}
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import TimelineVisualization from './components/TimelineVisualization.vue'

const visualizationRef = ref(null)
const tableData = ref([])

// Handle data from visualization component
function handleDataUpdated(data) {
  tableData.value = data
}

// Format date for display
function formatDate(date) {
  return date.toISOString().split('T')[0]
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

body {
  overflow-y: auto;
  overflow-x: hidden;
  font-family: Arial, sans-serif;
}

#app {
  width: 100vw;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

.data-table-section {
  flex: 1;
  padding: 2rem;
  background: #2a2a2a;
  color: #ffffff;
  overflow-x: auto;
}

.data-table-section h2 {
  margin-bottom: 1.5rem;
  color: #4ecdc4;
  font-size: 1.5rem;
  font-weight: 600;
}

.data-table {
  width: 100%;
  border-collapse: collapse;
  background: #1a1a1a;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.3);
}

.data-table thead {
  background: #333333;
}

.data-table th {
  padding: 1rem;
  text-align: left;
  font-weight: 600;
  color: #4ecdc4;
  border-bottom: 2px solid #444444;
}

.data-table tbody tr {
  border-bottom: 1px solid #333333;
  transition: background-color 0.2s;
}

.data-table tbody tr.clickable-row {
  cursor: pointer;
}

.data-table tbody tr:hover {
  background: #2d2d2d;
}

.data-table tbody tr:last-child {
  border-bottom: none;
}

.data-table td {
  padding: 1rem;
  color: #dddddd;
}

.color-badge {
  display: inline-block;
  width: 20px;
  height: 20px;
  border-radius: 4px;
  margin-right: 8px;
  vertical-align: middle;
  border: 1px solid #666;
}
</style>