<template>
  <div :style="backgroundStyle" class="widget-container">
    <div class="overlay" role="region" aria-live="polite" aria-labelledby="uv-index">
      <h1 id="uv-index" class="text-5xl font-bold mb-6">UV Index</h1>
      <div class="text-8xl font-extrabold mb-4" role="status">
        {{ uvIndex !== null ? uvIndex : '...' }}
      </div>
      <div class="text-2xl mb-6" aria-live="assertive">{{ spfAdvice }}</div>
      <div class="text-lg opacity-80">{{ weatherDescription }}</div>
      <div v-if="!latitude || !longitude" class="geolocation-info">
        <p>Для точных данных о погоде и уровне UV требуется доступ к вашему местоположению.</p>
        <button @click="getUserLocation" aria-label="Определить ваше местоположение">Определить местоположение</button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'

const uvIndex = ref<number | null>(null)
const weatherDescription = ref<string>('Loading...')
const backgroundStyle = ref<Record<string, string>>({})
const latitude = ref<number | null>(null)
const longitude = ref<number | null>(null)
let intervalId: number | null = null

const spfAdvice = computed(() => {
  if (uvIndex.value === null) return 'Loading...'
  if (uvIndex.value >= 3) return 'Наноси SPF!'
  return 'Можно без SPF.'
})

async function fetchWeatherAndUV() {
  if (latitude.value === null || longitude.value === null) return

  try {
    const response = await fetch(
        `https://api.open-meteo.com/v1/forecast?latitude=${latitude.value}&longitude=${longitude.value}&current=uv_index,weather_code`
    )
    const data = await response.json()

    uvIndex.value = data.current.uv_index

    const weatherCode = data.current.weather_code
    weatherDescription.value = mapWeatherCodeToDescription(weatherCode)

    // Определяем фон по погоде
    let bgUrl = ''
    if ([2, 3].includes(weatherCode)) {
      bgUrl = 'https://images.unsplash.com/photo-1502082553048-f009c37129b9?ixlib=rb-4.0.3&auto=format&fit=crop&w=1470&q=80'
    } else if ([61, 63, 65].includes(weatherCode)) {
      bgUrl = 'https://images.unsplash.com/photo-1527766833261-b09c3163a791?ixlib=rb-4.0.3&auto=format&fit=crop&w=1470&q=80'
    } else if (weatherCode === 0) {
      bgUrl = 'https://images.unsplash.com/photo-1502082553048-f009c37129b9?ixlib=rb-4.0.3&auto=format&fit=crop&w=1470&q=80'
    } else {
      bgUrl = 'https://images.unsplash.com/photo-1499346030926-9a72daac6c63?ixlib=rb-4.0.3&auto=format&fit=crop&w=1470&q=80'
    }

    backgroundStyle.value = {
      backgroundImage: `url('${bgUrl}')`,
      backgroundSize: 'cover',
      backgroundPosition: 'center',
      backgroundRepeat: 'no-repeat'
    }
  } catch (error) {
    console.error('Ошибка при загрузке данных:', error)
  }
}

function mapWeatherCodeToDescription(code: number): string {
  const mapping: Record<number, string> = {
    0: 'Ясно',
    1: 'Преимущественно ясно',
    2: 'Переменная облачность',
    3: 'Пасмурно',
    45: 'Туман',
    48: 'Туман',
    51: 'Морось',
    53: 'Морось',
    55: 'Морось',
    61: 'Небольшой дождь',
    63: 'Умеренный дождь',
    65: 'Сильный дождь',
    80: 'Ливень',
    81: 'Ливень',
    82: 'Сильный ливень',
    95: 'Гроза'
  }
  return mapping[code] || 'Неизвестная погода'
}

function getUserLocation() {
  if (!navigator.geolocation) {
    console.error('Геолокация не поддерживается вашим браузером.')
    return
  }

  navigator.geolocation.getCurrentPosition(
      (position) => {
        latitude.value = position.coords.latitude
        longitude.value = position.coords.longitude
        fetchWeatherAndUV()
      },
      (error) => {
        console.error('Ошибка при получении геолокации:', error)
      }
  )
}

onMounted(() => {
  getUserLocation()
  intervalId = window.setInterval(fetchWeatherAndUV, 60 * 60 * 1000) // Обновление раз в час
})

onUnmounted(() => {
  if (intervalId !== null) {
    clearInterval(intervalId)
  }
})
</script>

<style scoped>
.widget-container {
  position: relative;
  width: 300px;
  height: 400px;
  border-radius: 15px;
  box-shadow: 0 4px 10px rgb(14 144 115 / 48%);
  overflow: hidden;
  background-color: rgba(0, 0, 0, 0.6);
}

.overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5); /* Полупрозрачный фон */
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  padding: 20px;
  text-shadow: 2px 2px 6px rgba(0, 0, 0, 0.7); /* Тень для текста, чтобы улучшить читаемость */
}

h1, .text-8xl, .text-2xl, .text-lg {
  color: white;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.7);
}

html, body {
  height: 100%;
  margin: 0;
  font-family: 'Inter', sans-serif;
}

.geolocation-info {
  text-align: center;
  color: white;
  margin-top: 20px;
}

.geolocation-info button {
  background-color: #1c7ed6;
  color: white;
  padding: 10px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.geolocation-info button:focus {
  outline: 2px solid #ffb81c;
}
</style>
