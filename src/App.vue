<template>
  <div :style="backgroundStyle" class="widget-container">
    <div class="overlay">
      <div class="uvindex-card" role="region" aria-live="polite">
        <div class="uvindex-header">
          <img v-if="weatherIcon" :src="weatherIcon" alt="" class="weather-icon"/>
          <div class="uvindex-info">
            <h1 class="uvindex__label">UV Index</h1>
            <span class="uvindex__status" role="status">
              {{ uvIndex !== null ? uvIndex : "..." }}
              <span v-if="uvTrend" class="uvindex__trend">{{ uvTrend }}</span>
            </span>
            <span class="uvindex__advice">{{ spfAdvice }}</span>
          </div>
        </div>

        <div class="uvindex__description">
          {{ weatherDescription }}
        </div>

        <div v-if="showUpcomingUV && upcomingUV.length" class="uvindex-forecast">
          <h2>UV в ближайшие часы</h2>
          <ul>
            <li v-for="entry in upcomingUV" :key="entry.time">
              <time>{{ entry.time }}</time>
              <span class="uvindex-card__index">{{ entry.value.toFixed(1) }}</span>
            </li>
          </ul>
        </div>

        <div class="uvindex-footer">
          <button @click="fetchWeatherAndUV" class="refresh-button" aria-label="Обновить данные">
            Обновить
          </button>
          <span v-if="lastUpdatedText" class="uvindex__updated">
            {{ lastUpdatedText }}
          </span>
          <span v-if="cityName" class="uvindex__city">{{ cityName }}</span>
        </div>

        <div v-if="!latitude || !longitude" class="geolocation-info">
          <p>Для точных данных требуется доступ к вашему местоположению.</p>
          <button @click="getUserLocation" class="geo-button" aria-label="Определить ваше местоположение">
            Определить местоположение
          </button>
        </div>
      </div>
    </div>
  </div>
</template>


<script setup lang="ts">
import {ref, computed, onMounted, onUnmounted} from "vue"

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
  timeTicker.value // просто упоминаем, чтобы создать реактивную зависимость
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

const weatherOptions: Record<number, { description: string, backgroundGradient: string, icon: string }> = {
  0: {
    description: "Ясно",
    backgroundGradient: "linear-gradient(135deg, #f6d365 0%, #fda085 100%)",
    icon: "https://cdn-icons-png.flaticon.com/512/869/869869.png" // солнце
  },
  1: {
    description: "В основном ясно",
    backgroundGradient: "linear-gradient(135deg, #fceabb 0%, #f8b500 100%)",
    icon: "https://cdn-icons-png.flaticon.com/512/869/869869.png"
  },
  2: {
    description: "Переменная облачность",
    backgroundGradient: "linear-gradient(135deg, #d7d2cc 0%, #304352 100%)",
    icon: "https://cdn-icons-png.flaticon.com/512/1163/1163624.png" // облако + солнце
  },
  3: {
    description: "Пасмурно",
    backgroundGradient: "linear-gradient(135deg, #bdc3c7 0%, #2c3e50 100%)",
    icon: "https://cdn-icons-png.flaticon.com/512/414/414825.png" // облако
  },
  61: {
    description: "Небольшой дождь",
    backgroundGradient: "linear-gradient(135deg, #89f7fe 0%, #66a6ff 100%)",
    icon: "https://cdn-icons-png.flaticon.com/512/414/414974.png" // дождь
  },
  63: {
    description: "Умеренный дождь",
    backgroundGradient: "linear-gradient(135deg, #667db6 0%, #0082c8 50%, #0082c8 50%, #667db6 100%)",
    icon: "https://cdn-icons-png.flaticon.com/512/3313/3313983.png" // капли
  },
  65: {
    description: "Сильный дождь",
    backgroundGradient: "linear-gradient(135deg, #485563 0%, #29323c 100%)",
    icon: "https://cdn-icons-png.flaticon.com/512/3076/3076129.png" // ливень
  },
  95: {
    description: "Гроза",
    backgroundGradient: "linear-gradient(135deg, #2c3e50 0%, #000000 100%)",
    icon: "https://cdn-icons-png.flaticon.com/512/1146/1146869.png" // молния
  }
}

const weatherIcon = ref<string>("")

function getWeatherInfo(code: number) {
  const info = weatherOptions[code] ?? {
    description: "Неизвестная погода",
    backgroundGradient: "linear-gradient(135deg, #636363 0%, #a2ab58 100%)",
    icon: null
  }
  weatherIcon.value = info.icon
  return info
}

async function fetchCityName() {
  if (latitude.value === null || longitude.value === null) return

  try {
    const response = await fetch(
        `https://nominatim.openstreetmap.org/reverse?format=jsonv2&lat=${latitude.value}&lon=${longitude.value}`
    )
    const data = await response.json()

    cityName.value = data.address.city || data.address.town || data.address.village || "Неизвестный город"
  } catch (error) {
    console.error("Ошибка при получении города:", error)
  }
}

const cityName = ref<string>("")
const upcomingUV = ref<{ time: string; value: number }[]>([])

async function fetchWeatherAndUV() {
  if (latitude.value === null || longitude.value === null) return

  try {
    const response = await fetch(
        `https://api.open-meteo.com/v1/forecast?latitude=${latitude.value}&longitude=${longitude.value}&current=uv_index,weather_code&hourly=uv_index&timezone=auto`
    )
    const data = await response.json()

    uvIndex.value = +data.current.uv_index.toFixed(1)
    const weatherCode = data.current.weather_code
    const {description, backgroundGradient} = getWeatherInfo(weatherCode)

    weatherDescription.value = description
    backgroundStyle.value = {
      background: backgroundGradient
    }

    lastUpdatedAt.value = new Date()

    // Прогноз на ближайшие 4 часа
    // const now = new Date()
    // const hours = data.hourly.time.map((t: string) => new Date(t))
    const nowUtc = Date.now()
    upcomingUV.value = []

    for (let i = 0; i < data.hourly.time.length; i++) {
      const t = data.hourly.time[i];
      const hourUtcMs = new Date(t + 'Z').getTime();
      if (hourUtcMs  > nowUtc && upcomingUV.value.length < 4) {
        upcomingUV.value.push({
          time: new Date(hourUtcMs).toLocaleTimeString('ru-RU', {
            hour: '2-digit', minute: '2-digit',
            timeZone: data.timezone
          }),
          value: data.hourly.uv_index[i]
        });
      }
    }
  } catch (error) {
    console.error("Ошибка при загрузке данных:", error)
  }
}

const uvTrend = computed(() => {
  if (!upcomingUV.value.length) return null

  const firstUV = uvIndex.value
  const nextUV = upcomingUV.value[0].value
  if (firstUV) {
    if (nextUV > firstUV) {
      return "↑"
    } else if (nextUV < firstUV) {
      return "↓"
    } else {
      return "→"
    }
  }

})
const showUpcomingUV = computed(() => {
  const now = new Date()
  const hour = now.getHours()
  return hour >= 8 && hour <= 18
})

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
        fetchCityName()
      },
      (error) => {
        console.error("Ошибка при получении геолокации:", error)
      }
  )
}

const timeTicker = ref(0) // добавляем

onMounted(() => {
  getUserLocation()
  updateInterval = window.setInterval(fetchWeatherAndUV, 5 * 60 * 1000) // каждые 5 минут
  timeUpdateInterval = window.setInterval(() => {
    timeTicker.value++ // ТЕПЕРЬ тикер обновляется
  }, 30 * 1000)
})


onUnmounted(() => {
  if (updateInterval !== null) clearInterval(updateInterval)
  if (timeUpdateInterval !== null) clearInterval(timeUpdateInterval)
})
</script>


<style scoped>
/* Основной контейнер */
.widget-container {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100dvh;
  padding: 1rem;
  background: linear-gradient(135deg, #1e1e2f, #2e2e48);
  box-sizing: border-box;
  color: #fff;
  font-family: "Segoe UI", sans-serif;
  transition: background 0.5s ease;
}

.overlay {
  background-color: rgba(0, 0, 0, 0.45);
  border-radius: 1.5rem;
  backdrop-filter: blur(12px);
  padding: 2rem;
  max-width: 420px;
  width: 100%;
  box-shadow: 0 0 30px rgba(0, 0, 0, 0.25);
  animation: fadeInUp 0.6s ease;
}

@keyframes fadeInUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* Основная карточка */
.uvindex-card {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.uvindex-header {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.weather-icon {
  width: 64px;
  height: 64px;
  animation: rotateIn 0.8s ease;
}

@keyframes rotateIn {
  from {
    opacity: 0;
    transform: rotate(-20deg) scale(0.8);
  }
  to {
    opacity: 1;
    transform: rotate(0deg) scale(1);
  }
}

.uvindex-info {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.uvindex__label {
  font-size: 1.25rem;
  font-weight: bold;
}

.uvindex__status {
  font-size: 2.5rem;
  font-weight: bold;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  animation: popIn 0.4s ease;
}

@keyframes popIn {
  from {
    transform: scale(0.8);
    opacity: 0.5;
  }
  to {
    transform: scale(1);
    opacity: 1;
  }
}

.uvindex__trend {
  font-size: 1.5rem;
  opacity: 0.8;
}

.uvindex__advice {
  font-size: 1rem;
  font-style: italic;
  opacity: 0.9;
}

/* Описание погоды */
.uvindex__description {
  font-size: 1.125rem;
  opacity: 0.9;
  line-height: 1.4;
}

/* Прогноз */
.uvindex-forecast h2 {
  font-size: 1.1rem;
  margin-bottom: 0.5rem;
}

.uvindex-forecast ul {
  list-style: none;
  padding: 0;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.uvindex-forecast li {
  display: flex;
  justify-content: space-between;
  font-size: 1rem;
  background-color: rgba(255, 255, 255, 0.1);
  padding: 0.5rem 1rem;
  border-radius: 0.75rem;
  transition: background 0.3s;
}

.uvindex-forecast li:hover {
  background-color: rgba(255, 255, 255, 0.2);
}

.uvindex-footer {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  align-items: center;
  margin-top: 1rem;
}

/* Кнопки */
.refresh-button,
.geo-button {
  background-color: rgba(255, 255, 255, 0.15);
  border: none;
  border-radius: 0.75rem;
  padding: 0.5rem 1rem;
  color: #fff;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
}

.refresh-button:hover,
.geo-button:hover {
  background-color: rgba(255, 255, 255, 0.3);
  transform: scale(1.05);
}

.uvindex__updated,
.uvindex__city {
  font-size: 0.9rem;
  opacity: 0.85;
}

/* Геолокация */
.geolocation-info {
  margin-top: 1.5rem;
  text-align: center;
  font-size: 0.95rem;
  opacity: 0.9;
}

/* Адаптивность */
@media (max-width: 600px) {
  .overlay {
    padding: 1.25rem;
  }

  .weather-icon {
    width: 48px;
    height: 48px;
  }

  .uvindex__status {
    font-size: 2rem;
  }

  .uvindex__label {
    font-size: 1rem;
  }

  .uvindex__description {
    font-size: 1rem;
  }
}
</style>


