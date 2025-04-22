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
      <span v-if="cityName" class="uvindex__city">{{ cityName }}</span>
      <span class="uvindex__advice text-2xl mb-6" aria-live="assertive">{{ spfAdvice }}</span>
      <span class="uvindex__description text-lg opacity-80" aria-live="assertive">
      <img v-if="weatherIcon" :src="weatherIcon" alt="" class="weather-icon"/>
        {{ weatherDescription }}
      </span>

      <div v-if="!latitude || !longitude" class="geolocation-info">
        <p>Для точных данных о погоде и уровне UV требуется доступ к вашему местоположению.</p>
        <button @click="getUserLocation" aria-label="Определить ваше местоположение">Определить местоположение</button>
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
    icon: "https://cdn-icons-png.flaticon.com/512/414/414974.png" // дождик
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
    icon: "https://path-to-icons/unknown.svg"
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

async function fetchWeatherAndUV() {
  if (latitude.value === null || longitude.value === null) return

  try {
    const response = await fetch(
        `https://api.open-meteo.com/v1/forecast?latitude=${latitude.value}&longitude=${longitude.value}&current=uv_index,weather_code`
    )
    const data = await response.json()

    uvIndex.value = data.current.uv_index
    const weatherCode = data.current.weather_code
    const {description, backgroundGradient} = getWeatherInfo(weatherCode)

    weatherDescription.value = description
    backgroundStyle.value = {
      background: backgroundGradient
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
        fetchCityName()
      },
      (error) => {
        console.error("Ошибка при получении геолокации:", error)
      }
  )
}

onMounted(() => {
  getUserLocation()
  updateInterval = window.setInterval(fetchWeatherAndUV, 5 * 60 * 1000) // каждые 5 минут
  timeUpdateInterval = window.setInterval(() => {
  }, 30 * 1000) // триггерим пересчёт текста раз в 30 сек
})

onUnmounted(() => {
  if (updateInterval !== null) clearInterval(updateInterval)
  if (timeUpdateInterval !== null) clearInterval(timeUpdateInterval)
})
</script>


<style scoped>
@media (min-width: 768px) {
  .uvindex__label {
    font-size: 6rem; /* чуть меньше заголовок */
  }

  .uvindex__status {
    font-size: 20rem; /* крупно, но в пределах экрана */
  }

  .uvindex__advice, .uvindex__description {
    font-size: 4rem;
  }

  .refresh-button {
    font-size: 4rem;
    padding: 0.8rem 2rem;
  }

  .uvindex__updated {
    font-size: 3rem;
  }
}

@media (min-width: 1024px) {
  .overlay {
    justify-content: center;
  }
}

.geolocation-info button,
.refresh-button {
  min-height: 44px;
  min-width: 100px;
  font-size: 1rem;
}


.uvindex {
  color: white;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.7);
}

.uvindex__label {
  font-size: 4rem;
  line-height: 0.8;

}

.uvindex__status {
  font-size: 10rem;
  line-height: 1;
  margin-bottom: 1rem;
}

.uvindex__label {
  color: white;
}

.uvindex__advice, .uvindex__description {
  font-size: 2.5rem;
}

.uvindex__description {
  color: #d1d1d1;
}

.widget-container {
  position: relative;
  width: 100%;
  border-radius: 15px;
  box-shadow: 0px 2px 16px rgb(39 100 247 / 70%);
  overflow: hidden;
  background-color: rgba(0, 0, 0, 0.6);
}

.overlay {
  height: 100vh !important;
  background: rgba(0, 0, 0, 0.5); /* Полупрозрачный фон */
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
  text-shadow: 2px 2px 6px rgb(39 100 247 / 70%); /* Тень для текста, чтобы улучшить читаемость */
}

html, body {
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
  outline: 2px solid #1c7ed6;
}

.refresh-button {
  background: transparent;
  border: 0.25rem solid white;
  padding: 1rem 3rem;
  font-size: 2rem;
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
  font-size: 1.2rem;
  color: #d8d8d8;
}

.uvindex__city {
  color: #d8d8d8;
  font-size: 2rem;
  text-align: center;
  line-height: 0.9;
}

.uvindex__advice {
  color: white;
}

.weather-icon {
  width: 2rem;
  height: 2rem;
  vertical-align: middle;
  margin-right: 0.5rem;
}
</style>
