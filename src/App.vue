<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from "vue";
import { toPng } from "html-to-image";

// Phone element ref for screenshot
const phoneRef = ref(null);
const isGenerating = ref(false);

// Config Refs (Persisted)
const avatarImage = ref(''); // base64 data URL
const avatarFileInput = ref(null);
const username = ref('用户名');
const timeMode = ref('今天'); // '今天', '昨天', '自定义'
const customDate = ref('');
const customTime = ref('12:00');
const distMin = ref(2);
const distMax = ref(2.5);
const paceMin = ref(4);
const paceMax = ref(8);
const city = ref('北京市');

// App State Refs (Generated/Real-time)
const currentTime = ref('17:00');
const currentBattery = ref(10);
const activityDate = ref('2024/09/10');
const startTime = ref('09:48');
const endTime = ref('09:52');
const weather = ref('晴');
const temperature = ref('25°C');
const activityType = ref('户外跑步');
const distance = ref('2.28');
const distanceUnit = ref('公里');

const trainingDuration = ref('00:04:19');
const avgPace = ref("15'21''");
const calories = ref('24');
const totalDuration = ref('00:04:21');
const exerciseLoad = ref('1');
const avgCadence = ref('93');
const avgStride = ref('0.69');
const avgPower = ref('0');
const avgGroundContactTime = ref('0');

// --- 状态栏随机化：各元素独立随机组合 ---
const carriers = ['中国移动', '中国联通', '中国电信'];

const pick = (arr) => arr[Math.floor(Math.random() * arr.length)];

const randomPhone = () => {
  // 信号条：随机起始高度 + 递增步长
  const base = 3 + Math.random() * 2;        // 3–5
  const step = 2.5 + Math.random() * 1;      // 2.5–3.5
  const signalBars = [0, 1, 2, 3].map(i => Math.round(base + step * i));

  const barWidth = pick([2.5, 3, 3, 3.5]);
  const barGap = pick([1, 1, 1.5, 1.5, 2]);
  const barRadius = pick([0, 0.5, 0.5, 1, 1]);

  // 连接类型：15% WiFi，其余 4G/5G
  const isWifi = Math.random() < 0.15;
  const showWifi = isWifi;
  const networkType = isWifi ? '' : pick(['4G', '5G', '5G']);

  // 运营商：20% 显示在左侧
  const hasCarrier = Math.random() < 0.2;
  const carrierPosition = hasCarrier ? 'left' : 'none';
  const carrier = hasCarrier ? pick(carriers) : '';

  // 电池样式独立随机
  const batteryShowBorder = Math.random() < 0.7;
  const batteryPercentPos = pick(['none', 'none', 'outside', 'outside', 'inside']);
  const batteryPercentSign = batteryPercentPos !== 'none' ? Math.random() < 0.6 : false;

  // 时间字体
  const timeWeight = pick([500, 600, 600]);
  const timeSize = pick(['14px', '15px', '15px']);

  // 手机高度小幅随机
  const phoneHeight = Math.round(790 + Math.random() * 30); // 790–820

  // 底部栏：35% 小白条, 30% 导航栏, 35% 无
  const bottomBarRand = Math.random();
  let bottomBarType, navbarStyle, indicatorWidth;
  if (bottomBarRand < 0.35) {
    bottomBarType = 'indicator';
    indicatorWidth = Math.round(100 + Math.random() * 40); // 100–140px
    navbarStyle = 0;
  } else if (bottomBarRand < 0.65) {
    bottomBarType = 'navbar';
    navbarStyle = pick([1, 2, 3]); // 3种导航栏样式
    indicatorWidth = Math.round(100 + Math.random() * 40);
    } else {
    bottomBarType = 'none';
    navbarStyle = 0;
    indicatorWidth = 0;
  }

  return { signalBars, barWidth, barGap, barRadius, networkType, carrierPosition, carrier, batteryShowBorder, batteryPercentPos, batteryPercentSign, showWifi, timeWeight, timeSize, phoneHeight, bottomBarType, navbarStyle, indicatorWidth };
};

const currentPhone = ref(randomPhone());
const signalStrength = ref(4); // 1-4 格信号，控制哪些条亮/暗

// 地图背景随机偏移
const mapOffsetX = ref(0);
const mapOffsetY = ref(0);

// 跑步轨迹生成（操场区域，基于 480x360 图片坐标）
function generateRoute() {
  const cx = 243, cy = 180;
  const halfWidth = 70;      // 半圆半径（= 跑道半宽）
  const straightHalf = 30;   // 直道半长
  const straightLen = 2 * straightHalf;
  const semiArc = Math.PI * halfWidth;
  const perimeter = 2 * straightLen + 2 * semiArc;
  const startParam = Math.random();
  const laps = 6 + Math.random() * 2;
  const steps = Math.round(laps * 24);
  const points = [];
  for (let i = 0; i <= steps; i++) {
    const param = (startParam + (i / steps) * laps) % 1;
    const dist = param * perimeter;
    let x, y;
    if (dist < straightLen) {
      // 段1：右直道（从上到下）
      x = cx + halfWidth;
      y = (cy - straightHalf) + dist;
    } else if (dist < straightLen + semiArc) {
      // 段2：下半圆（从右到左，向下弯曲）
      const d = dist - straightLen;
      const angle = (d / semiArc) * Math.PI; // 0 → π
      x = cx + halfWidth * Math.cos(angle);
      y = (cy + straightHalf) + halfWidth * Math.sin(angle);
    } else if (dist < 2 * straightLen + semiArc) {
      // 段3：左直道（从下到上）
      const d = dist - straightLen - semiArc;
      x = cx - halfWidth;
      y = (cy + straightHalf) - d;
    } else {
      // 段4：上半圆（从左到右，向上弯曲）
      const d = dist - 2 * straightLen - semiArc;
      const angle = Math.PI + (d / semiArc) * Math.PI; // π → 2π
      x = cx + halfWidth * Math.cos(angle);
      y = (cy - straightHalf) + halfWidth * Math.sin(angle);
    }
    // 每个点独立随机偏移 ±3px
    const jx = (Math.random() - 0.5) * 6;
    const jy = (Math.random() - 0.5) * 6;
    points.push([Math.round((x + jx) * 10) / 10, Math.round((y + jy) * 10) / 10]);
  }
  return points;
}

const routePoints = ref(generateRoute());
const routeColorPhases = ref([0, 0, 0]);

const routeSegments = computed(() => {
  const pts = routePoints.value;
  if (pts.length < 2) return [];
  const ph = routeColorPhases.value;
  const segs = [];
  const n = pts.length - 1;
  for (let i = 0; i < n; i++) {
    const t = i / n; 
    const shift = 0.5 * Math.sin(t * Math.PI * 6 + ph[0])
                + 0.3 * Math.sin(t * Math.PI * 14 + ph[1])
                + 0.2 * Math.sin(t * Math.PI * 23 + ph[2]);
    const r = Math.round(Math.max(0, Math.min(255, 36 + shift * 70)));
    const g = Math.round(Math.max(0, Math.min(255, 199 - shift * 30)));
    const b = Math.round(Math.max(0, Math.min(255, 137 - shift * 70)));
    const color = `rgb(${r},${g},${b})`;
    segs.push({ x1: pts[i][0], y1: pts[i][1], x2: pts[i+1][0], y2: pts[i+1][1], color });
  }
  return segs;
});
const routeStart = computed(() => routePoints.value[0]);
const routeEnd = computed(() => routePoints.value[routePoints.value.length - 1]);

// 电池图标内部填充宽度
const batteryFillWidth = computed(() => {
  const clamped = Math.max(0, Math.min(100, currentBattery.value));
  const maxWidth = currentPhone.value.batteryShowBorder ? 20 : 23;
  return (clamped / 100) * maxWidth;
});

// 电池颜色
const batteryColor = computed(() => {
  if (currentBattery.value <= 20) return '#FF3B30';
  if (currentBattery.value <= 50) return '#FF9500';
  return '#34C759';
});

// 电池百分比文本
const batteryPercentText = computed(() => {
  const val = currentBattery.value;
  return currentPhone.value.batteryPercentSign ? `${val}%` : `${val}`;
});

// 底部栏高度（用于偏移 bottom-action）
const bottomBarHeight = computed(() => {
  if (currentPhone.value.bottomBarType === 'indicator') return 20;
  if (currentPhone.value.bottomBarType === 'navbar') return 36;
  return 0;
});

// --- New Logic Below ---

const getRandom = (min, max) => min + Math.random() * (max - min);
const padZero = (n) => n.toString().padStart(2, '0');

// Avatar image upload with resize to keep localStorage small
const handleAvatarUpload = (event) => {
  const file = event.target.files[0];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = (e) => {
    const img = new Image();
    img.onload = () => {
      const MAX = 128;
      let w = img.width, h = img.height;
      if (w > MAX || h > MAX) {
        if (w > h) { h = Math.round(h * MAX / w); w = MAX; }
        else { w = Math.round(w * MAX / h); h = MAX; }
      }
      const canvas = document.createElement('canvas');
      canvas.width = w;
      canvas.height = h;
      canvas.getContext('2d').drawImage(img, 0, 0, w, h);
      avatarImage.value = canvas.toDataURL('image/jpeg', 0.8);
      saveConfig();
    };
    img.src = e.target.result;
  };
  reader.readAsDataURL(file);
};

const mapWeatherCode = (code) => {
  if (code === 0) return '晴';
  if (code >= 1 && code <= 3) return '多云';
  if (code >= 45 && code <= 48) return '雾';
  if (code >= 51 && code <= 67) return '雨';
  if (code >= 71 && code <= 77) return '雪';
  if (code >= 80 && code <= 82) return '阵雨';
  if (code >= 95 && code <= 99) return '雷暴';
  return '未知';
};

const generateRecord = async () => {
  isGenerating.value = true;
  let dateObj = new Date();
  
  if (timeMode.value === '今天') {
    dateObj = new Date();
    dateObj.setHours(12, Math.floor(Math.random() * 60), Math.floor(Math.random() * 60));
  } else if (timeMode.value === '昨天') {
    dateObj = new Date();
    dateObj.setDate(dateObj.getDate() - 1);
    dateObj.setHours(12, Math.floor(Math.random() * 60), Math.floor(Math.random() * 60));
  } else {
    dateObj = new Date(customDate.value || new Date());
    const [h, m] = customTime.value.split(':');
    dateObj.setHours(parseInt(h || 12), parseInt(m || 0), Math.floor(Math.random() * 60));
  }
  
  const yyyy = dateObj.getFullYear();
  const mm = padZero(dateObj.getMonth() + 1);
  const dd = padZero(dateObj.getDate());
  activityDate.value = `${yyyy}/${mm}/${dd}`;
  const apiDate = `${yyyy}-${mm}-${dd}`;
  
  const stH = padZero(dateObj.getHours());
  const stM = padZero(dateObj.getMinutes());
  startTime.value = `${stH}:${stM}`;
  
  const dist = getRandom(distMin.value, distMax.value);
  distance.value = dist.toFixed(2);
  
  const pace = getRandom(paceMin.value, paceMax.value);
  const paceMinPart = Math.floor(pace);
  const paceSecPart = Math.floor((pace - paceMinPart) * 60);
  avgPace.value = `${paceMinPart}'${padZero(paceSecPart)}''`;
  
  const durationSec = dist * pace * 60;
  const tH = Math.floor(durationSec / 3600);
  const tM = Math.floor((durationSec % 3600) / 60);
  const tS = Math.floor(durationSec % 60);
  trainingDuration.value = `${padZero(tH)}:${padZero(tM)}:${padZero(tS)}`;
  
  const endDateObj = new Date(dateObj.getTime() + durationSec * 1000);
  endTime.value = `${padZero(endDateObj.getHours())}:${padZero(endDateObj.getMinutes())}`;
  
  const totalSec = durationSec + Math.floor(Math.random() * 120);
  const ttH = Math.floor(totalSec / 3600);
  const ttM = Math.floor((totalSec % 3600) / 60);
  const ttS = Math.floor(totalSec % 60);
  totalDuration.value = `${padZero(ttH)}:${padZero(ttM)}:${padZero(ttS)}`;
  
  calories.value = Math.round(dist * 60 * (pace / 5)).toString();
  exerciseLoad.value = Math.round(dist * 2).toString();
  
  const cadence = Math.round(getRandom(150, 185));
  avgCadence.value = cadence.toString();
  
  const pace_in_seconds = pace * 60;
  avgStride.value = (1000 / (pace_in_seconds * cadence / 60)).toFixed(2);
  avgPower.value = Math.round(getRandom(150, 350)).toString();
  avgGroundContactTime.value = Math.round(getRandom(200, 300)).toString();
  currentBattery.value = Math.round(getRandom(5, 100));

  // 随机组合手机状态栏样式
  currentPhone.value = randomPhone();
  // 信号强度：70% 满格，20% 三格，10% 两格
  const rand = Math.random();
  signalStrength.value = rand < 0.7 ? 4 : (rand < 0.9 ? 3 : 2);

  mapOffsetX.value = Math.round(Math.random() * 40 - 20);
  mapOffsetY.value = Math.round(Math.random() * 40 - 20);
  routePoints.value = generateRoute();
  routeColorPhases.value = [
    Math.random() * Math.PI * 2,
    Math.random() * Math.PI * 2,
    Math.random() * Math.PI * 2,
  ];

  try {
    const isPast = new Date(apiDate) < new Date(new Date().setHours(0,0,0,0));
    const baseUrl = isPast ? '/weather/v1/archive' : '/weather/v1/era5';
    const url = `${baseUrl}?latitude=39.9042&longitude=116.4074&daily=temperature_2m_max,temperature_2m_min,weathercode&timezone=Asia/Shanghai&start_date=${apiDate}&end_date=${apiDate}`;
    
    const res = await fetch(url);
    const data = await res.json();
    if (data && data.daily) {
      const code = data.daily.weathercode[0];
      const tMax = data.daily.temperature_2m_max[0];
      const tMin = data.daily.temperature_2m_min[0];
      weather.value = mapWeatherCode(code);
      temperature.value = `${Math.round((tMax + tMin) / 2)}°C`;
    }
  } catch (err) {
    console.error("Weather API failed:", err);
    weather.value = '晴';
    temperature.value = '25°C';
  }
  isGenerating.value = false;
};

const saveConfig = () => {
  const cfg = {
    avatarImage: avatarImage.value,
    username: username.value,
    timeMode: timeMode.value,
    customDate: customDate.value,
    customTime: customTime.value,
    distMin: distMin.value,
    distMax: distMax.value,
    paceMin: paceMin.value,
    paceMax: paceMax.value,
    city: city.value
  };
  localStorage.setItem('keep_fake_config', JSON.stringify(cfg));
};

const loadConfig = () => {
  const saved = localStorage.getItem('keep_fake_config');
  if (saved) {
    try {
      const cfg = JSON.parse(saved);
      if (cfg.avatarImage !== undefined) avatarImage.value = cfg.avatarImage;
      if (cfg.username !== undefined) username.value = cfg.username;
      if (cfg.timeMode !== undefined) timeMode.value = cfg.timeMode;
      if (cfg.customDate !== undefined) customDate.value = cfg.customDate;
      if (cfg.customTime !== undefined) customTime.value = cfg.customTime;
      if (cfg.distMin !== undefined) distMin.value = cfg.distMin;
      if (cfg.distMax !== undefined) distMax.value = cfg.distMax;
      if (cfg.paceMin !== undefined) paceMin.value = cfg.paceMin;
      if (cfg.paceMax !== undefined) paceMax.value = cfg.paceMax;
      if (cfg.city !== undefined) city.value = cfg.city;
    } catch (e) {}
  }
};

watch([avatarImage, username, timeMode, customDate, customTime, distMin, distMax, paceMin, paceMax, city], () => {
  saveConfig();
}, { deep: true });

let clockTimer;
const updateClock = () => {
  const now = new Date();
  currentTime.value = `${padZero(now.getHours())}:${padZero(now.getMinutes())}`;
};

const downloadScreenshot = async () => {
  if (!phoneRef.value) return;
  try {
    const dataUrl = await toPng(phoneRef.value, { cacheBust: true, pixelRatio: 2 });
    const link = document.createElement('a');
    link.download = `Keep_Record_${Date.now()}.png`;
    link.href = dataUrl;
    link.click();
  } catch (err) {
    console.error('Screenshot failed:', err);
  }
};

onMounted(() => {
  loadConfig();
  if (!customDate.value) {
    const today = new Date();
    customDate.value = `${today.getFullYear()}-${padZero(today.getMonth()+1)}-${padZero(today.getDate())}`;
  }
  generateRecord();
  updateClock();
  clockTimer = setInterval(updateClock, 1000);
});

onUnmounted(() => {
  clearInterval(clockTimer);
});
</script>

<template>
  <div class="app-container">
    <div class="main" ref="phoneRef" :style="{ height: currentPhone.phoneHeight + 'px' }">
      <!-- Loading overlay -->
      <div v-if="isGenerating" class="loading-overlay">
        <div class="loading-spinner"></div>
        <div class="loading-text">生成中...</div>
      </div>
      <div class="header">
        <!-- 左侧：时间 + 可选运营商 -->
        <div class="header-left">
          <span class="time" :style="{ fontWeight: currentPhone.timeWeight, fontSize: currentPhone.timeSize }">{{ currentTime }}</span>
          <span v-if="currentPhone.carrierPosition === 'left'" class="carrier-name">{{ currentPhone.carrier }}</span>
        </div>

        <!-- 右侧：信号 + 网络/WiFi + 电池 -->
        <div class="status-right">
          <!-- 信号强度条 -->
          <div class="signal-bars" :style="{ gap: currentPhone.barGap + 'px' }">
            <div v-for="(h, i) in currentPhone.signalBars" :key="i" class="signal-bar"
                 :style="{
                   height: h + 'px',
                   width: currentPhone.barWidth + 'px',
                   borderRadius: currentPhone.barRadius + 'px',
                   opacity: i < signalStrength ? 1 : 0.3
                 }"></div>
          </div>

          <!-- WiFi 图标 -->
          <svg v-if="currentPhone.showWifi" class="wifi-icon" width="15" height="12" viewBox="0 0 16 12" fill="#1d1d1f">
            <path d="M8 11.5l-2.5-3c.7-.6 1.5-.9 2.5-.9s1.8.3 2.5.9L8 11.5z"/>
            <path d="M3.7 6.3c1.1-1.1 2.6-1.8 4.3-1.8s3.2.7 4.3 1.8l-1.3 1.3c-.8-.8-1.8-1.2-3-1.2s-2.2.4-3 1.2L3.7 6.3z"/>
            <path d="M1 4.2C2.7 2.5 5.2 1.4 8 1.4s5.3 1.1 7 2.8l-1.4 1.4C12.2 4.2 10.2 3.3 8 3.3S3.8 4.2 2.4 5.6L1 4.2z"/>
          </svg>

          <!-- 网络类型 (4G/5G) -->
          <span v-if="currentPhone.networkType && !currentPhone.showWifi" class="network-type">{{ currentPhone.networkType }}</span>

          <!-- 电池图标 (SVG) -->
          <svg class="battery-icon" width="27" height="12" viewBox="0 0 27 12">
            <!-- 电池外壳（有边框时显示） -->
            <rect v-if="currentPhone.batteryShowBorder" x="0.5" y="0.5" width="23" height="11" rx="2.5" ry="2.5"
                  fill="none" stroke="#1d1d1f" stroke-width="1" opacity="0.35" />
            <!-- 电池正极凸起 -->
            <rect x="24.5" y="3.5" width="2" height="5" rx="1" ry="1"
                  fill="#1d1d1f" opacity="0.2" />
            <!-- 电池填充（有边框：内缩；无边框：撑满外壳区域） -->
            <rect v-if="currentPhone.batteryShowBorder"
                  x="2" y="2" :width="batteryFillWidth" height="8" rx="1.5" ry="1.5"
                  :fill="batteryColor" />
            <rect v-else
                  x="0.5" y="0.5" :width="batteryFillWidth" height="11" rx="2.5" ry="2.5"
                  :fill="batteryColor" />
            <!-- 百分比在电池内部 -->
            <text v-if="currentPhone.batteryPercentPos === 'inside'"
                  x="12" y="6" text-anchor="middle" dominant-baseline="central"
                  fill="#1d1d1f" font-size="8" font-weight="600"
                  font-family="-apple-system, BlinkMacSystemFont, 'SF Pro Text', sans-serif">
              {{ batteryPercentText }}
            </text>
          </svg>

          <!-- 电池百分比在外部 -->
          <span v-if="currentPhone.batteryPercentPos === 'outside'" class="battery-percent">{{ batteryPercentText }}</span>
        </div>
      </div>
      
      <!-- Content Area -->
      <div class="keep-content">
        <!-- App Navigation Bar -->
        <div class="nav-bar">
          <div class="nav-left">
            <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
              <path d="M15 18L9 12L15 6" stroke-linecap="round" stroke-linejoin="round"/>
            </svg>
          </div>
          <div class="nav-center">
            <div class="privacy-badge">
              <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="#000" stroke-width="2">
                <rect x="3" y="11" width="18" height="11" rx="2" ry="2"/>
                <path d="M7 11V7a5 5 0 0110 0v4"/>
              </svg>
              <span>隐私</span>
            </div>
          </div>
          <div class="nav-right">
            <svg width="20" height="20" viewBox="0 0 22 22" fill="none" stroke="currentColor" stroke-width="2.5">
              <path d="M14 2h6v6" stroke-linecap="square" stroke-linejoin="miter"/>
              <path d="M19 3l-10 10" stroke-linecap="square"/>
              <path d="M18 12v8H2V4h8" stroke-linecap="square" stroke-linejoin="miter"/>
            </svg>
            <svg width="24" height="24" viewBox="0 0 24 24" fill="currentColor">
              <rect x="3" y="10" width="4" height="4" rx="0.5" />
              <rect x="10" y="10" width="4" height="4" rx="0.5" />
              <rect x="17" y="10" width="4" height="4" rx="0.5" />
            </svg>
          </div>
        </div>

        <div>
          <!-- User Info Section -->
          <div class="user-info">
            <div class="avatar">
              <img v-if="avatarImage" :src="avatarImage" class="avatar-img" />
              <span v-else>头</span>
            </div>
            <div class="user-details">
              <div class="username">{{ username }}</div>
              <div class="activity-meta">
                {{ activityDate }} {{ startTime }} - {{ endTime }} {{ city }} · {{ weather }} · {{ temperature }}
              </div>
            </div>
          </div>

          <!-- Activity Type + Distance -->
          <div class="activity-header">
            <div class="activity-type">
              <svg width="18" height="18" viewBox="0 0 1167 1280" fill="currentColor" class="run-icon">
                <g transform="translate(0,1280) scale(0.1,-0.1)">
                  <path d="M9270 12789 c-229 -29 -473 -124 -657 -256 -272 -194 -468 -494 -539 -825 -25 -114 -25 -369 -1 -491 94 -472 445 -861 918 -1021 167 -56 260 -70 459 -70 199 0 292 14 459 70 400 135 727 449 864 829 50 141 70 257 71 425 1 176 -11 259 -59 406 -165 505 -614 864 -1166 934 -96 12 -254 11 -349 -1z"/>
                  <path d="M5145 10889 c-80 -12 -193 -69 -252 -127 -42 -41 -1667 -2180 -1780 -2342 -24 -35 -54 -97 -69 -143 -23 -69 -27 -100 -27 -197 -1 -103 2 -124 29 -200 70 -200 212 -329 394 -362 128 -22 284 28 378 123 27 27 397 512 823 1078 l774 1030 443 0 443 1 -19 -27 c-11 -16 -769 -1026 -1684 -2246 l-1664 -2219 -1215 -1 -1214 -2 -68 -27 c-93 -38 -164 -87 -239 -166 -323 -343 -232 -974 171 -1186 134 -70 29 -66 1631 -66 1549 0 1541 0 1705 51 124 38 264 118 331 190 18 19 321 405 674 859 354 454 648 831 655 838 10 11 162 -154 768 -832 417 -465 757 -852 757 -860 0 -9 -140 -728 -311 -1598 -223 -1137 -312 -1611 -316 -1681 -21 -363 196 -688 508 -762 302 -71 612 129 718 465 11 36 180 886 376 1890 195 1004 376 1926 401 2049 40 196 45 239 46 355 0 197 -40 357 -124 486 -27 42 -731 859 -1325 1537 l-55 62 728 968 c400 532 728 969 729 971 1 1 211 -240 467 -535 256 -296 489 -560 517 -586 63 -59 178 -115 260 -127 144 -20 301 37 430 156 98 91 1610 1844 1649 1912 57 100 77 181 76 317 0 87 -5 137 -18 180 -49 163 -154 293 -292 361 -77 39 -79 39 -193 39 -104 -1 -122 -3 -174 -27 -32 -15 -77 -41 -100 -58 -24 -17 -328 -358 -677 -760 -349 -401 -638 -728 -641 -727 -4 1 -354 402 -780 891 -440 506 -796 907 -827 931 -30 24 -80 55 -111 71 -127 62 -38 58 -1401 60 -685 1 -1272 -2 -1305 -7z"/>
                </g>
              </svg>
              {{ activityType }}
            </div>
            <div class="distance-display">
              <span class="distance-number">{{ distance }}</span>
              <span class="distance-unit">{{ distanceUnit }}</span>
            </div>
          </div>

          <!-- Map Placeholder -->
          <div class="map-placeholder">
            <img
              src="./assets/backdrop.jpeg"
              class="map-bg"
              :style="{ transform: `translate(${mapOffsetX}px, ${mapOffsetY}px)` }"
            />
            <svg
              class="map-bg map-route-svg"
              viewBox="0 0 480 360"
              preserveAspectRatio="xMidYMid slice"
              :style="{ transform: `translate(${mapOffsetX}px, ${mapOffsetY}px)` }"
            >
              <line v-for="(seg, idx) in routeSegments" :key="idx"
                :x1="seg.x1" :y1="seg.y1" :x2="seg.x2" :y2="seg.y2"
                :stroke="seg.color" stroke-width="6" stroke-linecap="round"/>
              <circle :cx="routeStart[0]" :cy="routeStart[1]" r="7" fill="#24C789" stroke="white" stroke-width="2.5"/>
              <circle :cx="routeEnd[0]" :cy="routeEnd[1]" r="7" fill="#FF3B30" stroke="white" stroke-width="2.5"/>
            </svg>
            <div class="map-buttons">
              <div class="map-btn-row">
                <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="#333" stroke-width="2.5">
                  <path d="M3 8V3h5M16 3h5v5M21 16v5h-5M8 21H3v-5"/>
                </svg>
                <span class="map-btn-label">全屏</span>
              </div>
              <div class="map-btn-row">
                <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="#333" stroke-width="2.5">
                  <circle cx="12" cy="12" r="10"/>
                  <polygon points="10,7 18,12 10,17" fill="#333" stroke="none"/>
                </svg>
                <span class="map-btn-label">视频</span>
              </div>
            </div>
          </div>



          <!-- Section Header "运动数据" -->
          <div class="section-header">
            <div class="section-title">运动数据</div>
            <div class="section-more-btn">查看详情</div>
          </div>

          <!-- Data Grid -->
          <div class="data-grid">
            <div class="data-cell">
              <div class="data-label">训练时长</div>
              <div class="data-value">{{ trainingDuration }}</div>
            </div>
            <div class="data-cell">
              <div class="data-label">平均配速</div>
              <div class="data-value">{{ avgPace }}</div>
            </div>
            <div class="data-cell">
              <div class="data-label">运动消耗</div>
              <div class="data-value">{{ calories }}<span class="data-unit">千卡</span></div>
            </div>
            <div class="data-cell">
              <div class="data-label">总时长</div>
              <div class="data-value">{{ totalDuration }}</div>
            </div>
            <div class="data-cell">
              <div class="data-label">运动负荷</div>
              <div class="data-value">{{ exerciseLoad }}</div>
            </div>
            <div class="data-cell">
              <div class="data-label">平均步频</div>
              <div class="data-value">{{ avgCadence }}</div>
            </div>
            <div class="data-cell">
              <div class="data-label">平均功率</div>
              <div class="data-value">{{ avgPower }}<span class="data-unit">瓦</span></div>
            </div>
            <div class="data-cell">
              <div class="data-label">平均步幅</div>
              <div class="data-value">{{ avgStride }}<span class="data-unit">米</span></div>
            </div>
            <div class="data-cell">
              <div class="data-label">平均触地时间</div>
              <div class="data-value">{{ avgGroundContactTime }}<span class="data-unit">毫秒</span></div>
            </div>
          </div>
        </div>
        
        <!-- Bottom Action Button -->
        <div class="bottom-action" :style="{ bottom: bottomBarHeight + 'px' }">
          <button class="action-btn">
            <svg width="16" height="16" viewBox="0 0 24 24" fill="white" style="margin-right: 6px; transform: rotate(-45deg);">
              <path d="M2 21l21-9L2 3v7l15 2-15 2v7z"/>
            </svg>
            发布动态到社区
          </button>
        </div>
      </div>

      <!-- 底部栏：小白条 / 导航栏 / 无 -->
      <div v-if="currentPhone.bottomBarType === 'indicator'" class="bottom-bar-area">
        <div class="home-indicator" :style="{ width: currentPhone.indicatorWidth + 'px' }"></div>
      </div>
      <div v-else-if="currentPhone.bottomBarType === 'navbar'" class="bottom-bar-area bottom-navbar">
        <!-- 样式1：经典三键 ◁ ○ □ -->
        <template v-if="currentPhone.navbarStyle === 1">
          <svg width="16" height="16" viewBox="0 0 24 24"><polygon points="15,5 15,19 5,12" fill="#999"/></svg>
          <svg width="16" height="16" viewBox="0 0 24 24"><circle cx="12" cy="12" r="7" fill="none" stroke="#999" stroke-width="1.8"/></svg>
          <svg width="16" height="16" viewBox="0 0 24 24"><rect x="5" y="5" width="14" height="14" rx="2" fill="none" stroke="#999" stroke-width="1.8"/></svg>
        </template>
        <!-- 样式2：短横线三键 — ○ — -->
        <template v-if="currentPhone.navbarStyle === 2">
          <div class="navbar-line"></div>
          <svg width="16" height="16" viewBox="0 0 24 24"><circle cx="12" cy="12" r="7" fill="none" stroke="#999" stroke-width="1.8"/></svg>
          <div class="navbar-line"></div>
        </template>
        <!-- 样式3：手势导航条 -->
        <template v-if="currentPhone.navbarStyle === 3">
          <div class="nav-pill" :style="{ width: currentPhone.indicatorWidth + 'px' }"></div>
        </template>
      </div>
    </div>

    <!-- Configuration Panel -->
    <div class="config-panel">
      <h2 class="panel-title">运行记录配置</h2>
      <a class="author-link" href="https://www.tachyon.moe" target="_blank">作者：CyanTachyon（点击查看）</a>
      <a class="author-link" href="https://www.tachyon.moe/posts/pkus-fake-keep" target="_blank">项目说明（点击查看）</a>
      <div class="config-group">
        <label>头像</label>
        <div class="avatar-upload">
          <div class="avatar-preview" @click="$refs.avatarFileInput.click()">
            <img v-if="avatarImage" :src="avatarImage" class="avatar-preview-img" />
            <span v-else class="avatar-placeholder">+</span>
          </div>
          <input type="file" ref="avatarFileInput" accept="image/*" @change="handleAvatarUpload" style="display:none" />
          <span class="avatar-hint">点击上传</span>
        </div>
      </div>
      <div class="config-group">
        <label>用户名</label>
        <input type="text" v-model="username" />
      </div>
      <div class="config-group">
        <label>城市</label>
        <input type="text" v-model="city" />
      </div>
      <div class="config-group">
        <label>时间模式</label>
        <div class="radio-group">
          <label><input type="radio" value="今天" v-model="timeMode" /> 今天</label>
          <label><input type="radio" value="昨天" v-model="timeMode" /> 昨天</label>
          <label><input type="radio" value="自定义" v-model="timeMode" /> 自定义</label>
        </div>
      </div>
      <div class="config-group" v-if="timeMode === '自定义'">
        <label>自定义时间</label>
        <div class="split-inputs">
          <input type="date" v-model="customDate" />
          <input type="time" v-model="customTime" />
        </div>
      </div>
      <div class="config-group">
        <label>距离范围 (km)</label>
        <div class="split-inputs">
          <input type="number" step="0.1" v-model.number="distMin" placeholder="最小" />
          <span class="separator">-</span>
          <input type="number" step="0.1" v-model.number="distMax" placeholder="最大" />
        </div>
      </div>
      <div class="config-group">
        <label>配速范围 (min/km)</label>
        <div class="split-inputs">
          <input type="number" step="0.1" v-model.number="paceMin" placeholder="最快" />
          <span class="separator">-</span>
          <input type="number" step="0.1" v-model.number="paceMax" placeholder="最慢" />
        </div>
      </div>
      
      <div class="panel-actions">
        <button class="btn-primary" @click="generateRecord">重新生成</button>
        <button class="btn-secondary" @click="downloadScreenshot" :disabled="isGenerating">截图下载</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
@font-face {
  font-family: 'DIN Condensed Bold';
  src: url('@/assets/DINCond-Bold.otf') format('opentype');
  font-weight: bold;
  font-style: normal;
}

.app-container {
  display: flex;
  flex-direction: row;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  background-color: #f5f5f5;
  gap: 40px;
  padding: 40px 20px;
  box-sizing: border-box;
}

@media (max-width: 768px) {
  .app-container {
    flex-direction: column;
    padding: 20px 10px;
    gap: 24px;
  }
  .config-panel {
    width: 100% !important;
    max-width: 400px;
  }
}

@media (max-width: 440px) {
  .main {
    transform-origin: top center;
    transform: scale(calc((100vw - 20px) / 400));
    margin-bottom: calc(-800px * (1 - (100vw - 20px) / 400));
  }
}

/* Configuration Panel Styles */
.config-panel {
  width: 320px;
  background: #FFFFFF;
  border-radius: 16px;
  box-shadow: 0 4px 24px rgba(0, 0, 0, 0.08);
  padding: 24px;
  display: flex;
  flex-direction: column;
  gap: 20px;
  box-sizing: border-box;
  font-family: -apple-system, BlinkMacSystemFont, 'SF Pro Text', 'Helvetica Neue', sans-serif;
}

.panel-title {
  font-family: 'DIN Condensed Bold', sans-serif;
  font-size: 24px;
  margin: 0 0 0 0;
  color: #333;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.author-link {
  font-size: 12px;
  color: #999;
  text-decoration: none;
  margin-bottom: 4px;
}

.author-link:hover {
  color: #24C789;
}

.config-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.config-group label {
  font-size: 12px;
  font-weight: 600;
  color: #666;
  text-transform: uppercase;
}

.config-group input[type="text"],
.config-group input[type="number"],
.config-group input[type="date"],
.config-group input[type="time"] {
  padding: 10px 12px;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  font-size: 14px;
  outline: none;
  transition: border-color 0.2s;
  box-sizing: border-box;
}

.config-group input:focus {
  border-color: #24C789;
}

.split-inputs {
  display: flex;
  align-items: center;
  gap: 8px;
}

.split-inputs input {
  flex: 1;
  width: 0;
}

.separator {
  color: #999;
  font-weight: bold;
}

.radio-group {
  display: flex;
  gap: 16px;
  align-items: center;
  font-size: 14px;
}

.radio-group label {
  font-size: 14px;
  font-weight: normal;
  text-transform: none;
  color: #333;
  display: flex;
  align-items: center;
  gap: 4px;
  cursor: pointer;
}

.panel-actions {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-top: 10px;
}

.btn-primary, .btn-secondary {
  width: 100%;
  padding: 12px;
  border-radius: 24px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  border: none;
  transition: opacity 0.2s;
}

.btn-primary {
  background: #24C789;
  color: #fff;
  box-shadow: 0 4px 12px rgba(36, 199, 137, 0.3);
}

.btn-primary:active {
  opacity: 0.8;
}

.btn-secondary {
  background: #f0f0f0;
  color: #333;
}

.btn-secondary:active {
  background: #e0e0e0;
}


/* --- Original Phone App Styles Below --- */
.loading-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(255, 255, 255, 0.85);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  z-index: 100;
  gap: 12px;
}

.loading-spinner {
  width: 32px;
  height: 32px;
  border: 3px solid #e0e0e0;
  border-top-color: #24C789;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.loading-text {
  font-size: 14px;
  color: #666;
}

.btn-secondary:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.main {
  width: 400px;
  border: 1px solid #e0e0e0;
  border-radius: 0;
  overflow: hidden;
  background: #FFFFFF;
  box-shadow: 0 4px 24px rgba(0, 0, 0, 0.12);
  display: flex;
  flex-direction: column;
  position: relative;
  font-family: 'DIN Condensed Bold', -apple-system, BlinkMacSystemFont, 'SF Pro Text', 'Helvetica Neue', 'PingFang SC', sans-serif;
}

.header {
  height: 44px;
  flex-shrink: 0;
  background: #FFFFFF;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 24px;
  box-sizing: border-box;
  z-index: 10;
}

.header-left {
  display: flex;
  align-items: center;
  gap: 6px;
}

.time {
  font-size: 15px;
  font-weight: 600;
  font-family: -apple-system, BlinkMacSystemFont, 'SF Pro Text', 'Helvetica Neue', sans-serif;
  color: #1d1d1f;
  letter-spacing: 0.2px;
}

.carrier-name {
  font-size: 12px;
  font-weight: 400;
  font-family: -apple-system, BlinkMacSystemFont, 'SF Pro Text', 'Helvetica Neue', sans-serif;
  color: #1d1d1f;
  letter-spacing: 0.2px;
}

.status-right {
  display: flex;
  align-items: center;
  gap: 6px;
}

/* 信号格 */
.signal-bars {
  display: flex;
  align-items: flex-end;
  gap: 1.5px;
  height: 13px;
}

.signal-bar {
  width: 3px;
  background: #1d1d1f;
  border-radius: 1px;
}

/* 5G 标识 */
.network-type {
  font-size: 12px;
  font-weight: 600;
  font-family: -apple-system, BlinkMacSystemFont, 'SF Pro Text', 'Helvetica Neue', sans-serif;
  color: #1d1d1f;
  letter-spacing: 0.3px;
}

/* 电池百分比 */
.battery-percent {
  font-size: 12px;
  font-weight: 400;
  font-family: -apple-system, BlinkMacSystemFont, 'SF Pro Text', 'Helvetica Neue', sans-serif;
  color: #1d1d1f;
}

/* 电池图标 */
.battery-icon {
  display: block;
}

/* WiFi 图标 */
.wifi-icon {
  display: block;
}

/* --- Keep UI Styles --- */
.keep-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.nav-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 44px;
  padding: 0 16px;
  background: #FFFFFF;
  flex-shrink: 0;
}

.nav-left, .nav-right {
  display: flex;
  align-items: center;
  gap: 16px;
  color: #1d1d1f;
}

.nav-center {
  flex: 1;
  display: flex;
  justify-content: flex-end;
  padding-right: 16px;
}

.privacy-badge {
  display: flex;
  align-items: center;
  gap: 4px;
  background: #F2F2F2;
  padding: 4px 8px;
  border-radius: 12px;
  font-size: 11px;
  color: #000000;
}

.user-info {
  display: flex;
  align-items: center;
  padding: 12px 20px;
  gap: 12px;
}

.avatar {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  background: #24C789;
  color: #FFFFFF;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 16px;
  font-weight: 600;
  overflow: hidden;
  flex-shrink: 0;
}

.avatar-img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.avatar-upload {
  display: flex;
  align-items: center;
  gap: 12px;
}

.avatar-preview {
  width: 48px;
  height: 48px;
  border-radius: 50%;
  background: #f0f0f0;
  border: 2px dashed #ccc;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  overflow: hidden;
  flex-shrink: 0;
}

.avatar-preview-img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.avatar-placeholder {
  font-size: 24px;
  color: #999;
}

.avatar-hint {
  font-size: 12px;
  color: #999;
}

.user-details {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.username {
  font-size: 14px;
  font-weight: 400;
  color: #000000;
}

.activity-meta {
  font-size: 11px;
  color: #BDBDBD;
}

.activity-header {
  padding: 8px 20px 16px;
}

.activity-type {
  display: flex;
  align-items: center;
  gap: 4px;
  color: #000000;
  font-size: 18px;
  font-weight: 600;
  margin-bottom: 8px;
}

.distance-display {
  display: flex;
  align-items: baseline;
  gap: 4px;
}

.distance-number {
  font-size: 56px;
  font-weight: 1000;
  color: #000000;
  line-height: 1;
}

.distance-unit {
  font-size: 16px;
  color: #000000;
  font-weight: 400;
}

.map-placeholder {
  margin: 0 20px;
  height: 220px;
  background: #F5F5F5;
  border-radius: 0;
  position: relative;
  overflow: hidden;
}

.map-bg {
  position: absolute;
  top: -20px;
  left: -20px;
  width: calc(100% + 40px);
  height: calc(100% + 40px);
  object-fit: cover;
  pointer-events: none;
}

.map-buttons {
  position: absolute;
  right: 12px;
  top: 12px;
  display: flex;
  flex-direction: column;
  align-items: center;
  background: rgba(255, 255, 255, 0.92);
  border-radius: 0;
  box-shadow: 0 2px 8px rgba(0,0,0,0.08);
  padding: 6px 0;
}

.map-btn-row {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 2px;
  padding: 4px 10px;
  color: #333;
}

.map-btn-label {
  font-size: 10px;
  color: #666;
}


.section-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 24px 20px 16px;
}

.section-title {
  font-size: 18px;
  font-weight: 700;
  color: #000000;
}

.section-more-btn {
  font-size: 12px;
  color: #000000;
  background: #F5F5F5;
  padding: 4px 10px;
  border-radius: 12px;
}

.data-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 24px 16px;
  padding: 0 20px 24px;
}

.data-cell {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.data-label {
  font-size: 12px;
  color: #999999;
}

.data-value {
  font-size: 22px;
  font-weight: 700;
  color: #000000;
  display: flex;
  align-items: baseline;
  gap: 2px;
}

.data-unit {
  font-size: 12px;
  color: #999999;
  font-weight: 400;
}

.bottom-action {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  padding: 16px 28px;
  background: linear-gradient(to top, #FFFFFF 80%, rgba(255,255,255,0));
  display: flex;
  justify-content: center;
}

.action-btn {
  width: auto;
  padding: 0 48px;
  height: 48px;
  border-radius: 24px;
  background: #24C789;
  color: #FFFFFF;
  border: none;
  font-size: 16px;
  font-weight: 600;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  box-shadow: 0 4px 12px rgba(36, 199, 137, 0.3);
}

/* 底部栏通用容器 */
.bottom-bar-area {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #FFFFFF;
  z-index: 11;
}

/* 小白条 (iOS home indicator) */
.home-indicator {
  height: 5px;
  background: #1d1d1f;
  border-radius: 3px;
  margin: 8px 0;
}

/* 导航栏 */
.bottom-navbar {
  height: 36px;
  gap: 80px;
}

/* 导航栏样式2的短横线 */
.navbar-line {
  width: 14px;
  height: 2px;
  background: #999;
  border-radius: 1px;
}

/* 导航栏样式3的手势条 */
.nav-pill {
  height: 4px;
  background: #1d1d1f;
  border-radius: 2px;
  opacity: 0.4;
}
</style>