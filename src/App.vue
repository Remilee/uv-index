<template>
  <div :style="backgroundStyle" class="widget-container">
    <div class="uvindex overlay" role="region" aria-live="polite">
      <span class="uvindex__label">UV Index:</span>
      <span class="uvindex__status" role="status">
        {{ uvIndex ?? "..." }}
      </span>
      <button @click="fetchWeatherAndUV" class="refresh-button" aria-label="Обновить данные">
        Обновить
      </button>
      <span v-if="lastUpdatedText" class="uvindex__updated">
      {{ lastUpdatedText }}
      </span>
      <span class="uvindex__advice text-2xl mb-6" aria-live="assertive">{{ spfAdvice }}</span>
      <span class="uvindex__description text-lg opacity-80">{{ weatherDescription }}</span>

      <div v-if="!latitude || !longitude" class="geolocation-info">
        <p>Для точных данных о погоде и уровне UV требуется доступ к вашему местоположению.</p>
        <button @click="getUserLocation" aria-label="Определить ваше местоположение">Определить местоположение</button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from "vue"

const uvIndex = ref<number | null>(null)
const weatherDescription = ref("Loading...")
const backgroundStyle = ref<Record<string, string>>({})
const lastUpdatedAt = ref<Date | null>(null)
const latitude = ref<number | null>(null)
const longitude = ref<number | null>(null)

let updateInterval: number | null = null
let timeUpdateInterval: number | null = null

const spfAdvice = computed(() => {
  if (uvIndex.value === null) return "Loading..."
  return uvIndex.value >= 3 ? "Наноси SPF!" : "Можно без SPF"
})

const lastUpdatedText = computed(() => {
  if (!lastUpdatedAt.value) return ""

  const now = new Date()
  const diffMs = now.getTime() - lastUpdatedAt.value.getTime()
  const diffMinutes = Math.floor(diffMs / 60000)

  if (diffMinutes < 1) return "Обновлено только что"
  if (diffMinutes === 1) return "Обновлено 1 минуту назад"
  if (diffMinutes < 5) return `Обновлено ${diffMinutes} минуты назад`
  if (diffMinutes < 60) return `Обновлено ${diffMinutes} минут назад`

  const diffHours = Math.floor(diffMinutes / 60)
  if (diffHours === 1) return "Обновлено 1 час назад"
  return `Обновлено ${diffHours} часов назад`
})

const weatherOptions: Record<number, { description: string, backgroundUrl: string }> = {
  0: {
    description: "Ясно",
    backgroundUrl: "https://images.unsplash.com/photo-1502082553048-f009c37129b9?ixlib=rb-4.0.3&auto=format&fit=crop&w=1470&q=80"
  },
  2: {
    description: "Переменная облачность",
    backgroundUrl: "https://images.unsplash.com/photo-1502082553048-f009c37129b9?ixlib=rb-4.0.3&auto=format&fit=crop&w=1470&q=80"
  },
  3: {
    description: "Пасмурно",
    backgroundUrl: "https://images.unsplash.com/photo-1502082553048-f009c37129b9?ixlib=rb-4.0.3&auto=format&fit=crop&w=1470&q=80"
  },
  61: {
    description: "Небольшой дождь",
    backgroundUrl: "https://images.unsplash.com/photo-1527766833261-b09c3163a791?ixlib=rb-4.0.3&auto=format&fit=crop&w=1470&q=80"
  },
  63: {
    description: "Умеренный дождь",
    backgroundUrl: "https://images.unsplash.com/photo-1527766833261-b09c3163a791?ixlib=rb-4.0.3&auto=format&fit=crop&w=1470&q=80"
  },
  65: {
    description: "Сильный дождь",
    backgroundUrl: "https://images.unsplash.com/photo-1527766833261-b09c3163a791?ixlib=rb-4.0.3&auto=format&fit=crop&w=1470&q=80"
  },
  95: {
    description: "Гроза",
    backgroundUrl: "https://images.unsplash.com/photo-1499346030926-9a72daac6c63?ixlib=rb-4.0.3&auto=format&fit=crop&w=1470&q=80"
  }
}

function getWeatherInfo(code: number) {
  return weatherOptions[code] ?? {
    description: "Неизвестная погода",
    backgroundUrl: "https://images.unsplash.com/photo-1499346030926-9a72daac6c63?ixlib=rb-4.0.3&auto=format&fit=crop&w=1470&q=80"
  }
}

async function fetchWeatherAndUV() {
  if (latitude.value === null || longitude.value === null) return

  try {
    const response = await fetch(
        `https://api.open-meteo.com/v1/forecast?latitude=${latitude.value}&longitude=${longitude.value}&current=uv_index,weather_code`
    )
    const data = await response.json()

    uvIndex.value = data.current.uv_index
    const weatherCode = data.current.weather_code
    const { description, backgroundUrl } = getWeatherInfo(weatherCode)

    weatherDescription.value = description
    backgroundStyle.value = {
      backgroundImage: `url('${backgroundUrl}')`,
      backgroundSize: "cover",
      backgroundPosition: "center",
      backgroundRepeat: "no-repeat"
    }

    lastUpdatedAt.value = new Date()
  } catch (error) {
    console.error("Ошибка при загрузке данных:", error)
  }
}

function getUserLocation() {
  if (!navigator.geolocation) {
    console.error("Геолокация не поддерживается вашим браузером.")
    return
  }

  navigator.geolocation.getCurrentPosition(
      (position) => {
        latitude.value = position.coords.latitude
        longitude.value = position.coords.longitude
        fetchWeatherAndUV()
      },
      (error) => {
        console.error("Ошибка при получении геолокации:", error)
      }
  )
}

onMounted(() => {
  getUserLocation()
  updateInterval = window.setInterval(fetchWeatherAndUV, 5 * 60 * 1000) // каждые 5 минут
  timeUpdateInterval = window.setInterval(() => {}, 30 * 1000) // триггерим пересчёт текста раз в 30 сек
})

onUnmounted(() => {
  if (updateInterval !== null) clearInterval(updateInterval)
  if (timeUpdateInterval !== null) clearInterval(timeUpdateInterval)
})
</script>


<style scoped>
/* Оставляем твои стили без изменений */
.uvindex {
  color: white;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.7);
}

.uvindex__label {
  font-size: 4vw;
}

.uvindex__status {
  font-size: 13vw;
  line-height: normal;
}

.uvindex__label {
  color: white;
}

.uvindex__advice, .uvindex__description {
  font-size: 2.5vw;
}

.uvindex__description {
  color: #d1d1d1;
}

.widget-container {
  position: relative;
  width: 100%;
  height: 100vh;
  border-radius: 15px;
  box-shadow: 0px 2px 16px rgb(39 100 247 / 70%);
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
  text-shadow: 2px 2px 6px rgb(39 100 247 / 70%); /* Тень для текста, чтобы улучшить читаемость */
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

.refresh-button {
  background: transparent;
  border: 0.25vw solid white;
  padding: 1rem 3rem;
  font-size: 2vw;
  border-radius: 4rem;
  cursor: pointer;
  color: white;
  margin: 10px 0;
  transition: background-color 0.2s ease;
}

.refresh-button:hover {
  background-color: rgba(204, 204, 204, 0.45);
  transition: background-color 0.2s ease;
}

.refresh-button:focus {
  background-color: rgba(204, 204, 204, 0.45);
  transition: background-color 0.2s ease;
}
.uvindex__updated {
  margin-top: 10px;
  font-size: 1.5vw;
  color: #cccccc;
}
</style>
