<template>
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
          v-for="(item, index) in data"
          :key="'data-' + index"
          @click="handleRowClick(item)"
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

    <h2 v-if="events.length > 0" style="margin-top: 2rem;">Events</h2>
    <table v-if="events.length > 0" class="data-table">
      <thead>
        <tr>
          <th>Date</th>
          <th>Title</th>
          <th>Description</th>
        </tr>
      </thead>
      <tbody>
        <tr
          v-for="(event, index) in events"
          :key="'event-' + index"
          class="event-row"
        >
          <td>{{ formatDate(event.timestamp) }}</td>
          <td>{{ event.title }}</td>
          <td>{{ event.description }}</td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script setup>
import { defineProps, defineEmits } from 'vue'

const props = defineProps({
  data: {
    type: Array,
    required: true,
    default: () => []
  },
  events: {
    type: Array,
    default: () => []
  }
})

const emit = defineEmits(['row-click'])

// Format date for display
function formatDate(date) {
  return date.toISOString().split('T')[0]
}

// Handle row click
function handleRowClick(item) {
  emit('row-click', item)
}
</script>

<style scoped>
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

.event-row {
  background: #1f1f1f;
}

.event-row:hover {
  background: #2d2d2d;
}
</style>
