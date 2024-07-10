<script setup lang="ts">
import { ref, onMounted } from 'vue'
import axios from 'axios'

interface Location {
  name: string
  country: string
  latitude: number
  longitude: number
}

const cityName = ref<string>('Berlin')
const resultCount = ref<number>(10)

const locations = ref<Location[]>([])
const error = ref<string | null>(null)

const fetchLocations = async () => {
  try {
    const response = await axios.get('https://geocoding-api.open-meteo.com/v1/search', {
      params: {
        name: cityName.value,
        count: resultCount.value,
        language: 'en',
        format: 'json'
      }
    })
    locations.value = response.data.results
  } catch (err) {
    error.value =
      axios.isAxiosError(err) && err.response ? err.response.data : 'An unknown error occurred'
  }
}

onMounted(fetchLocations)
</script>

<template>
  <main>
    <div>
      <label
        >City name:
        <input v-model="cityName" type="text" />
      </label>
      <label>
        Number of results:
        <select v-model="resultCount">
          <option value="1">1</option>
          <option value="10">10</option>
          <option value="20">20</option>
          <option value="50">50</option>
          <option value="100">100</option>
        </select>
      </label>
      <button @click="fetchLocations">Search</button>
    </div>
    <div v-if="error">{{ error }}</div>
    <ul v-else>
      <li v-for="location in locations" :key="location.latitude + location.longitude">
        {{ location.name }}, {{ location.country }} (Lat: {{ location.latitude }}, Lon:
        {{ location.longitude }} )
      </li>
    </ul>
  </main>
</template>
