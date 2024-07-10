<script setup lang="ts">
import { ref, onMounted } from 'vue'
import axios from 'axios'
import { format, parseISO } from 'date-fns'

const dataLoaded = ref<boolean>(false)

interface Location {
  name: string
  country: string
  latitude: number
  longitude: number
  weather: {
    forecast: {
      temperature: number[]
      time: string[]
    }

    currentConditions: string
  }
  showForecast: boolean
}

const cityName = ref<string>('Berlin')
const resultCount = ref<number>(10)
const daysCount = ref<number>(3)

const locations = ref<Location[]>([])
const error = ref<string | null>(null)

const formattedLocations = ref<
  {
    cityName: string
    countryName: string
    latitude: number
    longitude: number
    currentWeather: string
    forecast: { [key: string]: [number, string][] }
    showForecast: boolean
  }[]
>([])

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
              current: 'temperature_2m',
              start_date: format(today, 'yyyy-MM-dd'),
              end_date: format(endDay, 'yyyy-MM-dd')
            }
          })

          location.weather = {
            forecast: {
              temperature: weatherResponse.data.hourly.temperature_2m,
              time: weatherResponse.data.hourly.time
            },
            currentConditions: weatherResponse.data.current.temperature_2m
          }
        } catch (err) {
          console.error('Failed to fetch weather data for location:', location.name, err)
          location.weather = {
            forecast: { temperature: [], time: [] },
            currentConditions: 'N/A'
          }
        }
      })
    )

    formattedLocations.value = locations.value.map((loc) => {
      const location = {
        cityName: loc.name,
        countryName: loc.country,
        latitude: loc.latitude,
        longitude: loc.longitude,
        currentWeather: loc.weather.currentConditions,
        forecast: {} as { [key: string]: [number, string][] },
        showForecast: false
      }

      if (loc.weather && typeof loc.weather.forecast !== 'string') {
        loc.weather.forecast.time.forEach((date, index) => {
          const formattedDate = formatTime(date)[0]
          if (!location.forecast[formattedDate]) {
            location.forecast[formattedDate] = []
          }
          const temperature = loc.weather.forecast.temperature[index]
          console.log(loc.weather.forecast.temperature)
          const time = formatTime(loc.weather.forecast.time[index])[1]
          location.forecast[formattedDate].push([temperature, time])
        })
      }

      return location
    })
    dataLoaded.value = true
  } catch (err) {
    error.value =
      axios.isAxiosError(err) && err.response ? err.response.data : 'An unknown error occurred'
    dataLoaded.value = true
  }
}

const formatTime = (utcTime: string) => {
  const date = parseISO(utcTime) // Parsing ISO date
  const dateTime = date.getTime() // Extracting time value
  const dateTimezoneOffset = date.getTimezoneOffset() // Retreiving timezone offset from GMT
  const dateTimezoneOffsetMilliseconds = dateTimezoneOffset * 60000 // Calculating offset in milliseconds
  const localTime = new Date(dateTime - dateTimezoneOffsetMilliseconds) // Calculating local time

  return [format(localTime, 'MMM do'), format(localTime, 'HH:mm')]
}

const toggleForecast = (location: Location) => {
  location.showForecast = !location.showForecast
}

onMounted(fetchLocations)
</script>

<template>
  <main>
    <div v-if="dataLoaded === true">
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
      <div v-if="error">{{ error }}dadada</div>
      <ul v-else>
        <li v-for="location in formattedLocations" :key="location.latitude + location.longitude">
          <h3>{{ location.cityName }}, {{ location.countryName }}</h3>
          <div v-if="location.currentWeather">
            <p>Current Conditions: {{ location.currentWeather }} °C</p>
            <button @click="toggleForecast(location as unknown as Location)">
              {{ location.showForecast ? 'Hide forecast' : 'Show forecast' }}
            </button>
            <div v-if="location.showForecast">
              <div v-for="(temperatures, date) in location.forecast" :key="date">
                <h4>{{ date }}</h4>
                <ul>
                  <li v-for="[temperature, time] in temperatures" :key="time">
                    {{ time }} - {{ temperature }} °C
                  </li>
                </ul>
              </div>
            </div>
          </div>
        </li>
      </ul>
    </div>
    <div v-else>
      loading...
      <div>
        <b-spinner class="m-5" label="Busy"></b-spinner>
      </div>
    </div>
  </main>
</template>
