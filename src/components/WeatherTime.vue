<template>
  <!-- 天气时钟 -->
  <div
    :class="[
      'weather-time',
      status.siteStatus,
      status.mainBoxBig && status.siteStatus !== 'normal' && status.siteStatus !== 'focus'
        ? 'hidden'
        : null,
      set.showLunar ? 'lunar' : null,
      set.timeStyle,
    ]"
    @click.stop
  >
    <div
      class="time"
      @click.stop="
        status.setSiteStatus(
          status.siteStatus !== 'normal' && status.siteStatus !== 'focus' ? 'normal' : 'box',
        )
      "
    >
      <span class="hour">{{ timeData.hour ?? "00" }}</span>
      <span class="separator" :key="set.showSeconds">:</span>
      <span class="minute">{{ timeData.minute ?? "00" }}</span>
      <Transition name="fade" mode="out-in">
        <span v-if="set.showSeconds" class="second">
          <span class="separator">:</span>
          <span class="second-num">{{ timeData.second ?? "00" }}</span>
        </span>
      </Transition>
      <template v-if="set.use12HourFormat">
        <span class="amPm">{{ timeData.amPm ?? "am" }}</span>
      </template>
    </div>
    <div v-if="set.showLunar" class="lunar">
      <span class="year">{{ timeData.lunar?.GanZhiYear }}</span>
      <span class="text">{{ timeData.lunar?.text }}</span>
    </div>
    <div class="date">
      <span class="month">{{ timeData.month ?? "0" }}</span>
      <span class="day">{{ timeData.day ?? "0" }}</span>
      <span class="weekday">{{ timeData.weekday ?? "星期八" }}</span>
    </div>
    <div v-if="set.showWeather && weatherAvailable && weatherData" class="weather">
      <span class="status">{{ weatherData.condition }}</span>
      <span class="temperature">{{ weatherData.temp }} ℃</span>
      <span class="wind">{{ weatherData.windDir }}</span>
      <span v-if="weatherData.windLevel" class="wind-level">
        {{ weatherData.windLevel }} 级
      </span>
    </div>
  </div>
</template>

<script setup>
import { getCurrentTime } from "@/utils/timeTools";
import { ref, onMounted, onBeforeUnmount, watch } from "vue";
import { statusStore, setStore } from "@/stores";
import { getAdcode, getWeather } from "@/api";

const set = setStore();
const status = statusStore();

// 时间数据
const timeData = ref({});
const timeInterval = ref(null);

// 天气数据
const weatherData = ref(null);
const weatherAvailable = ref(true);
const weatherKey = import.meta.env.VITE_WEATHER_KEY;

// 更新时间
const updateTimeData = () => {
  timeData.value = getCurrentTime(set.showZeroTime, set.use12HourFormat);
};

// 获取城市 adcode
// 高德 /v3/ip 接口对部分个人 Key 会返回空 adcode（province/city/adcode 均为 []），
// 此时直接查天气会得到 INVALID_PARAMS(20000)，故取不到 adcode 时回退到默认城市。
const DEFAULT_ADCODE = "110000"; // 北京

const getCityAdcode = async () => {
  try {
    const result = await getAdcode(weatherKey);
    if (result.infocode !== "10000") {
      console.warn("IP 定位失败：", result.info, "，回退到默认城市");
      return DEFAULT_ADCODE;
    }
    // 高德在无法定位时可能返回空数组，需做有效性校验
    const adcode = Array.isArray(result.adcode) ? result.adcode[0] : result.adcode;
    if (!adcode) {
      console.warn("IP 定位返回空 adcode（个人 Key 常见），回退到默认城市");
      return DEFAULT_ADCODE;
    }
    return adcode;
  } catch (error) {
    console.warn("IP 定位请求异常，回退到默认城市：", error);
    return DEFAULT_ADCODE;
  }
};

// 获取天气数据
const getWeatherData = async () => {
  if (!weatherKey) {
    // 未配置 Key 时静默隐藏天气区域，不打扰用户
    console.warn("未配置天气 Key，已隐藏天气显示");
    weatherAvailable.value = false;
    return;
  }
  // 当前时间戳
  const currentTime = Date.now();
  // 上次获取天气数据的数据
  let lastWeatherData = JSON.parse(localStorage.getItem("lastWeatherData")) || {
    data: {},
    lastFetchTime: 0,
  };
  // 上次获取天气数据的时间戳与当前时间的时间差（毫秒）
  const timeDifference = currentTime - lastWeatherData.lastFetchTime;
  // 是否超出 5 分钟
  if (timeDifference >= 5 * 60 * 1000) {
    const adcode = await getCityAdcode();
    const weatherResult = await getWeather(weatherKey, adcode);
    if (weatherResult.infocode !== "10000") {
      // 天气接口失败时静默降级，不再弹错误提示
      console.error("天气查询失败：", weatherResult.info, weatherResult.infocode);
      weatherAvailable.value = false;
      return;
    }
    const data = weatherResult.lives?.[0];
    if (!data) {
      console.error("天气数据为空，已隐藏天气显示");
      weatherAvailable.value = false;
      return;
    }
    weatherAvailable.value = true;
    weatherData.value = {
      condition: data.weather,
      temp: data.temperature,
      windDir: data.winddirection + "风",
      windLevel: data.windpower,
    };
    lastWeatherData = { data: weatherData.value, lastFetchTime: currentTime };
    // 储存新天气数据
    localStorage.setItem("lastWeatherData", JSON.stringify(lastWeatherData));
  } else {
    console.log("从缓存中读取天气数据：", lastWeatherData);
    weatherData.value = lastWeatherData.data;
    weatherAvailable.value = Boolean(lastWeatherData.data);
  }
};

// 监听配置发生改变
watch(
  () => [set.showZeroTime, set.use12HourFormat],
  () => {
    updateTimeData();
  },
);

onMounted(() => {
  // 时间
  updateTimeData();
  timeInterval.value = setInterval(updateTimeData, 1000);
  // 天气
  getWeatherData();
});

onBeforeUnmount(() => {
  clearInterval(timeInterval.value);
});
</script>

<style lang="scss" scoped>
.weather-time {
  position: absolute;
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-bottom: 20px;
  transform: translateY(-140px);
  color: var(--main-text-color);
  animation: fade-time-in 0.6s cubic-bezier(0.21, 0.78, 0.36, 1);
  transition:
    transform 0.3s,
    opacity 0.5s,
    margin-bottom 0.3s;
  z-index: 1;
  .time {
    cursor: pointer;
    font-size: 3rem;
    margin: 6px 0px;
    text-shadow: var(--main-text-shadow);
    transition: transform 0.3s;
    .separator {
      opacity: 0.8;
      font-size: 2.8rem;
      display: inline-block;
      margin: 0 5px;
      transform: translateY(-4px);
      animation: separator-breathe 0.7s infinite alternate;
    }
    .amPm {
      font-size: 1rem;
      opacity: 0.6;
      margin-left: 6px;
    }
    &:hover {
      transform: scale(1.08);
    }
    &:active {
      transform: scale(1);
    }
  }
  .date {
    font-size: 1.15rem;
    opacity: 0.8;
    margin: 4px 0px;
    text-shadow: var(--main-text-shadow);
    .month {
      &::after {
        margin: 0 4px;
        content: "月";
      }
    }
    .day {
      &::after {
        margin: 0 8px 0 4px;
        content: "日";
      }
    }
  }
  .lunar {
    font-size: 0.9rem;
    opacity: 0.6;
    text-shadow: var(--main-text-shadow);
    .year {
      &::after {
        margin-right: 4px;
        content: "年";
      }
    }
  }
  .weather {
    opacity: 0.7;
    font-size: 1rem;
    text-shadow: var(--main-text-shadow);
    .temperature {
      margin: 0 6px;
    }
    .wind-level {
      margin-left: 6px;
    }
  }

  &.focus {
    transform: translateY(-180px);
    // transform: translateY(-24vh);
  }
  &.box,
  &.set {
    // transform: translateY(-220px);
    transform: translateY(-34vh);
    @media (max-width: 478px) {
      transform: translateY(-32vh);
    }
  }
  &.hidden {
    transform: translateY(-180px);
    // transform: translateY(-24vh);
    opacity: 0;
  }
  &.lunar {
    margin-bottom: 50px;
  }
  &.two {
    padding-bottom: 60px;
    .time {
      display: flex;
      flex-direction: column;
      align-items: center;
      span {
        line-height: normal;
      }
      .separator,
      .second {
        display: none;
      }
      .hour {
        &::after {
          content: "/";
          font-size: 2rem;
          display: flex;
          align-items: center;
          justify-content: center;
          line-height: 0;
          opacity: 0.4;
          transform: rotate(50deg);
          margin: 12px 0;
        }
      }
    }
  }
}
</style>
