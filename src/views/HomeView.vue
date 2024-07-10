<script setup lang="ts">
import { ref, onMounted } from 'vue'
import axios from 'axios'
import { format, parseISO } from 'date-fns'

interface Location {
  name: string
  country: string
  latitude: number
  longitude: number
  weather?: {
    forecast:
      | {
          temperature: []
          time: []
        }
      | string

    currentConditions: string
  }
}

const cityName = ref<string>('Berlin')
const resultCount = ref<number>(10)
const daysCount = ref<number>(5)

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

    const today = new Date()
    const endDay = new Date()
    endDay.setDate(today.getDate() + daysCount.value - 1)

    await Promise.all(
      locations.value.map(async (location) => {
        try {
          const weatherResponse = await axios.get('https://api.open-meteo.com/v1/forecast', {
            params: {
              latitude: location.latitude,
              longitude: location.longitude,
              hourly: 'temperature_2m',
              start_date: format(today, 'yyyy-MM-dd'),
              end_date: format(endDay, 'yyyy-MM-dd')
            }
          })

          console.log(weatherResponse)

          location.weather = {
            forecast: {
              temperature: weatherResponse.data.hourly.temperature_2m,
              time: weatherResponse.data.hourly.time
            },
            currentConditions: weatherResponse.data.currentConditions
          }
        } catch (err) {
          console.error('Failed to fetch weather data for location:', location.name, err)
          location.weather = {
            forecast: 'N/A',
            currentConditions: 'N/A'
          }
        }
      })
    )
  } catch (err) {
    error.value =
      axios.isAxiosError(err) && err.response ? err.response.data : 'An unknown error occurred'
  }
}

const formatTime = (utcTime: string) => {
  const date = parseISO(utcTime) // Parsing ISO date
  const dateTime = date.getTime() // Extracting time value
  const dateTimezoneOffset = date.getTimezoneOffset() // Retreiving timezone offset from GMT
  const dateTimezoneOffsetMilliseconds = dateTimezoneOffset * 60000 // Calculating offset in milliseconds
  const localTime = new Date(dateTime - dateTimezoneOffsetMilliseconds) // Calculating local time

  return format(localTime, 'yyyy-MM-dd HH:mm')
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
      <label>
        Number of days:
        <select v-model="daysCount">
          <option :value="1">1</option>
          <option :value="2">2</option>
          <option :value="3">3</option>
          <option :value="4">4</option>
          <option :value="5">5</option>
        </select>
      </label>
      <button @click="fetchLocations">Search</button>
    </div>
    <div v-if="error">{{ error }}</div>
    <ul v-else>
      <li v-for="location in locations" :key="location.latitude + location.longitude">
        <h3>{{ location.name }}, {{ location.country }}</h3>
        <div v-if="location.weather">
          <div v-if="typeof location.weather.forecast === 'string'">
            <p>Current Conditions: {{ location.weather.currentConditions }}</p>
            <p>
              Forecast:
              {{ location.weather.forecast }}
            </p>
          </div>
          <table v-else>
            <p>Current Conditions: {{ location.weather.currentConditions }}</p>

            <tbody>
              <tr
                v-for="(temperature, index) in location.weather.forecast.temperature"
                :key="index"
              >
                <td>{{ formatTime(location.weather.forecast.time[index]) }}</td>
                <!-- <td>{{ format(location.weather.forecast.time[index], 'yyyy-MM-dd HH:mm') }}</td> -->
                <td>{{ temperature }} °C</td>
              </tr>
            </tbody>
          </table>
        </div>
      </li>
    </ul>
  </main>
</template>
