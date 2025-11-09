<template>
  <section class="voice-gallery" aria-label="珂莱塔 · 语音馆">
    <div class="bg-carousel" aria-hidden="true">
      <transition-group name="bg-fade" tag="div" class="bg-layer">
        <img
          v-for="(src, idx) in activeImages"
          :key="`bg-${idx}-${isMobile ? 'm' : 'd'}`"
          :src="src"
          :class="['bg-img', { active: idx === currentIndex }]"
          alt=""
        />
      </transition-group>
    </div>

    <div class="vg__wrap">
      <header class="vg__header">
        <div class="logo">
          <div class="title-group">
            <h1 class="title">长离 · 语音馆</h1>
            <p class="subtitle">局未终，人仍在。</p>
          </div>
        </div>
      </header>

      <!-- 列表（已解锁放前，未解锁放后） -->
      <ul class="vg__list" role="list">
        <li
          v-for="id in allVoiceIds"
          :key="id"
          class="vg__item"
          :class="{
            unlocked: isUnlocked(id),
            locked: !isUnlocked(id),
            playing: id === currentId,
          }"
        >
          <div class="item__left">
            <div class="index">{{ String(id).padStart(3, "0") }}</div>
            <div class="info">
              <div class="name">语音 {{ String(id).padStart(3, "0") }}</div>
              <div class="note" v-if="isUnlocked(id)">已解锁</div>
              <div class="note note--locked" v-else>未解锁</div>
            </div>
          </div>

          <div class="item__right">
            <button
              class="btn btn--icon"
              :disabled="!isUnlocked(id)"
              @click="playEntry(id)"
              :title="
                isUnlocked(id)
                  ? currentId === id && isPlaying
                    ? '暂停'
                    : '播放'
                  : '尚未解锁'
              "
            >
              <span v-if="!isUnlocked(id)">🔒</span>
              <span v-else-if="currentId === id && isPlaying">❚❚</span>
              <span v-else>▶</span>
            </button>

            <a
              v-if="isUnlocked(id)"
              :href="voicePath(id)"
              :download="`audio_${id}.mp3`"
              title="下载"
            >
              <el-button type="primary" :icon="Download" circle />
            </a>
            <span v-else class="btn btn--hint" aria-hidden="true">—</span>
          </div>
        </li>
      </ul>
    </div>
  </section>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onBeforeUnmount, watch } from "vue";
import { Download } from "@element-plus/icons-vue";
/* ================== 配置 ================== */
const TOTAL_VOICES = 30; // 语音总数，按实际替换
const BG_INTERVAL_MS = 4500; // 背景切换间隔（毫秒）
const MOBILE_BREAKPOINT = 720; // 小于这个宽度视为移动端
/* ========================================= */

/* ========== 导入图片资源（Vite：eager） ========== */
/* 横图（用于 PC） */
const modules: Record<string, any> = import.meta.glob(
  "@/assets/images1/*.{jpg,png,jpeg,webp}",
  { eager: true }
);
const allSrcs: string[] = Object.values(modules).map(
  (m: any) => m.default || m
);

/* 竖图（用于移动端） */
const modules2: Record<string, any> = import.meta.glob(
  "@/assets/images2/*.{jpg,png,jpeg,webp}",
  { eager: true }
);
const allSrcs2: string[] = Object.values(modules2).map(
  (m: any) => m.default || m
);

/* 简单洗牌函数 */
function shuffle<T>(arr: T[]) {
  const a = arr.slice();
  for (let i = a.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [a[i], a[j]] = [a[j], a[i]];
  }
  return a;
}

/* 随机取 5 张（若不足 5 张则全部） */
const randomFive = ref<string[]>(
  shuffle(allSrcs).slice(0, Math.min(5, allSrcs.length))
);
const randomFive2 = ref<string[]>(
  shuffle(allSrcs2).slice(0, Math.min(5, allSrcs2.length))
);

/* 轮播索引（共享，但 index 会根据 activeImages 长度做取模） */
const currentIndex = ref(0);
let imgTimer: number | null = null;

/* ========== 设备判断（响应式） ========== */
const isMobile = ref(window.innerWidth < MOBILE_BREAKPOINT);
function handleResize() {
  const nowMobile = window.innerWidth < MOBILE_BREAKPOINT;
  if (nowMobile !== isMobile.value) {
    isMobile.value = nowMobile;
    // 切换图片组时重置索引以避免越界
    currentIndex.value = 0;
  }
}

/* activeImages 根据 isMobile 返回对应数组 */
const activeImages = computed(() =>
  isMobile.value ? randomFive2.value : randomFive.value
);
/* ========== 语音列表与播放逻辑 ========== */

/* 已解锁集合（由 localStorage.triggeredVoices 提供，数组） */
const unlockedSet = ref<Set<number>>(new Set<number>());

function loadUnlocked() {
  try {
    const raw = localStorage.getItem("triggeredVoices") || "[]";
    const arr = JSON.parse(raw);
    const s = new Set<number>();
    if (Array.isArray(arr)) {
      arr.forEach((v: any) => {
        const n = Number(v);
        if (!Number.isNaN(n)) s.add(n);
      });
    }
    unlockedSet.value = s;
  } catch (e) {
    console.warn("读取 triggeredVoices 失败", e);
    unlockedSet.value = new Set<number>();
  }
}

/* 生成所有 id，并保持已解锁在前、未解锁在后 */
const allIds = Array.from({ length: TOTAL_VOICES }, (_, i) => i);
const allVoiceIds = computed(() => {
  const unlocked = Array.from(unlockedSet.value)
    .filter((n) => allIds.includes(n))
    .sort((a, b) => a - b);
  const locked = allIds.filter((id) => !unlockedSet.value.has(id));
  return [...unlocked, ...locked];
});

/* audio 单例 */
let currentAudio: HTMLAudioElement | null = null;
const currentId = ref<number | null>(null);
const isPlaying = ref(false);
const currentTime = ref(0);
const currentDuration = ref(0);

function createAudio(src?: string) {
  destroyAudio();
  currentAudio = new Audio();
  currentAudio.preload = "auto";
  if (src) currentAudio.src = src;
  currentAudio.addEventListener("timeupdate", onTimeUpdate);
  currentAudio.addEventListener("loadedmetadata", onLoadedMeta);
  currentAudio.addEventListener("ended", onEnded);
  currentAudio.addEventListener("error", onAudioError);
}
function destroyAudio() {
  if (!currentAudio) return;
  try {
    currentAudio.pause();
    currentAudio.removeEventListener("timeupdate", onTimeUpdate);
    currentAudio.removeEventListener("loadedmetadata", onLoadedMeta);
    currentAudio.removeEventListener("ended", onEnded);
    currentAudio.removeEventListener("error", onAudioError);
    currentAudio.src = "";
  } catch (e) {}
  currentAudio = null;
}
function onTimeUpdate() {
  if (currentAudio) currentTime.value = currentAudio.currentTime || 0;
}
function onLoadedMeta() {
  if (currentAudio) currentDuration.value = currentAudio.duration || 0;
}
function onEnded() {
  isPlaying.value = false; /* 不自动下一条 */
}
function onAudioError(e?: any) {
  console.error("audio error", e);
  isPlaying.value = false;
}

function voicePath(id: number) {
  return `/voice/audio (${id}).mp3`;
}
function isUnlocked(id: number) {
  return unlockedSet.value.has(id);
}

async function playEntry(id: number) {
  if (!isUnlocked(id)) return;
  // 同一条 -> 切换暂停/恢复
  if (currentId.value === id && isPlaying.value) {
    currentAudio?.pause();
    isPlaying.value = false;
    return;
  }
  if (currentId.value === id && !isPlaying.value && currentAudio) {
    try {
      await currentAudio.play();
      isPlaying.value = true;
    } catch (e) {
      console.warn(e);
    }
    return;
  }

  // 新条目
  currentId.value = id;
  createAudio(voicePath(id));
  try {
    await currentAudio!.play();
    isPlaying.value = true;
  } catch (e) {
    console.warn("播放被阻止或失败", e);
    isPlaying.value = false;
  }
}

/* ========== 背景轮播控制 ========== */
function startBgTimer() {
  stopBgTimer();
  imgTimer = window.setInterval(() => {
    const len = Math.max(1, activeImages.value.length);
    // 保证在当前 activeImages 长度范围内循环
    currentIndex.value = (currentIndex.value + 1) % len;
  }, BG_INTERVAL_MS);
}
function stopBgTimer() {
  if (imgTimer !== null) {
    clearInterval(imgTimer);
    imgTimer = null;
  }
}

/* 监听 storage 事件（跨 tab 更新） */
function onStorage(e: StorageEvent) {
  if (e.key === "triggeredVoices") loadUnlocked();
}

/* 生命周期 */
onMounted(() => {
  loadUnlocked();
  window.addEventListener("storage", onStorage);
  window.addEventListener("resize", handleResize);

  // 如果数组为空（没有图片），也避免报错：确保至少有一个占位空数组
  if (!randomFive.value.length && allSrcs.length)
    randomFive.value = shuffle(allSrcs).slice(0, Math.min(5, allSrcs.length));
  if (!randomFive2.value.length && allSrcs2.length)
    randomFive2.value = shuffle(allSrcs2).slice(
      0,
      Math.min(5, allSrcs2.length)
    );

  // 启动轮播
  startBgTimer();
});

onBeforeUnmount(() => {
  window.removeEventListener("storage", onStorage);
  window.removeEventListener("resize", handleResize);
  stopBgTimer();
  destroyAudio();
});

/* 监听 activeImages 长度变化（切换设备时重置 index 并保持循环） */
watch(activeImages, (nv) => {
  currentIndex.value = 0;
});
</script>

<style lang="scss" scoped>
/* 尤诺风格：月色冷蓝 / 银灰高光，轻微金色点缀（仅作点缀）*/
.voice-gallery {
  position: relative;
  min-height: 560px;
  font-family: "PingFang SC", "Noto Sans SC", system-ui, -apple-system,
    "Segoe UI", Roboto, "Helvetica Neue", Arial;
  color: #eaf7ff; /* 纸色 -> 冷白 */
  overflow: hidden;
  padding: 28px;
  padding-top: 80px;
  background: linear-gradient(
    180deg,
    #041028 0%,
    #021024 40%,
    #010313 100%
  ); /* 夜色底 */
  -webkit-font-smoothing: antialiased;
  /* 整体的月光气流层（轻微） */
  &::before {
    content: "";
    position: absolute;
    inset: 0;
    z-index: 0;
    pointer-events: none;
    background: radial-gradient(
        600px 200px at 12% 10%,
        rgba(95, 195, 255, 0.04),
        transparent 12%
      ),
      radial-gradient(
        420px 140px at 78% 18%,
        rgba(143, 217, 255, 0.02),
        transparent 12%
      );
    filter: blur(6px);
    mix-blend-mode: screen;
  }
  &::after {
    /* 微小星尘点缀 */
    content: "";
    position: absolute;
    inset: 0;
    z-index: 0;
    pointer-events: none;
    background-image: radial-gradient(
        circle at 8% 12%,
        rgba(223, 248, 255, 0.02) 0.6px,
        transparent 1px
      ),
      radial-gradient(
        circle at 84% 26%,
        rgba(255, 223, 138, 0.008) 0.6px,
        transparent 1px
      );
    background-size: 180px 180px, 220px 220px;
    mix-blend-mode: screen;
    opacity: 0.98;
  }

  /* 背景轮播层（冷色调 / 月光滤镜）*/
  .bg-carousel {
    position: absolute;
    inset: 0;
    z-index: 0;
    pointer-events: none;

    .bg-layer {
      position: absolute;
      inset: 0;
      overflow: hidden;

      .bg-img {
        position: absolute;
        left: 0;
        top: 0;
        width: 100%;
        height: 100%;
        object-fit: cover;
        display: block;
        opacity: 0;
        transform: scale(1.02);
        transition: opacity 900ms ease, transform 900ms ease, filter 900ms ease;
        pointer-events: none;
        /* 冷色滤镜 */
        filter: brightness(0.68) contrast(0.96) saturate(0.9) hue-rotate(-8deg);
        mix-blend-mode: screen;
      }

      .bg-img.active {
        opacity: 1;
        transform: scale(1);
        filter: brightness(0.92) contrast(1.02) saturate(1.04);
      }
    }
  }

  /* 前景容器（玻璃月匣感） */
  .vg__wrap {
    position: relative;
    z-index: 2;
    max-width: 980px;
    margin: 0 auto;
    border-radius: 14px;
    padding: 18px;
    box-shadow: 0 12px 48px rgba(2, 8, 18, 0.72),
      inset 0 1px 0 rgba(223, 248, 255, 0.02);
    background: linear-gradient(
      180deg,
      rgba(6, 10, 18, 0.6),
      rgba(4, 8, 14, 0.45)
    );
    border: 1px solid rgba(143, 217, 255, 0.03);
    backdrop-filter: blur(6px) saturate(1.02);
  }

  /* 头部 */
  .vg__header {
    display: flex;
    gap: 12px;
    align-items: center;
    margin-bottom: 24px;

    .logo {
      display: flex;
      gap: 12px;
      align-items: center;

      /* 月光呼吸动画 */
      @keyframes moonCoreBreathe {
        0% {
          transform: scale(1);
          opacity: 0.9;
          filter: drop-shadow(0 6px 18px rgba(78, 160, 255, 0.04));
        }
        50% {
          transform: scale(1.03);
          opacity: 1;
          filter: drop-shadow(0 18px 46px rgba(78, 160, 255, 0.08));
        }
        100% {
          transform: scale(1);
          opacity: 0.9;
          filter: drop-shadow(0 6px 18px rgba(78, 160, 255, 0.04));
        }
      }

      @keyframes starFloat {
        0% {
          opacity: 0;
          transform: translateY(0) scale(0.9);
          filter: blur(0);
        }
        35% {
          opacity: 1;
          transform: translateY(-6px) scale(1.05);
          filter: blur(0.2px);
        }
        70% {
          opacity: 0.6;
          transform: translateY(-10px) scale(1.15);
          filter: blur(0.8px);
        }
        100% {
          opacity: 0;
          transform: translateY(-14px) scale(1.25);
          filter: blur(1.6px);
        }
      }

      .title-group {
        display: flex;
        flex-direction: column;

        .title {
          margin: 0;
          font-size: 1.5rem;
          font-weight: 800;
          /* 冷蓝渐变文字 */
          background: linear-gradient(
            90deg,
            #9fe8ff 0%,
            #6fc9ff 50%,
            #2f92ff 100%
          );
          -webkit-background-clip: text;
          background-clip: text;
          color: transparent;
          -webkit-text-fill-color: transparent;
          text-shadow: 0 6px 20px rgba(2, 8, 18, 0.6);
          letter-spacing: 0.4px;
        }

        .subtitle {
          margin: 4px 0 0;
          color: rgba(223, 248, 255, 0.82);
          font-size: 1rem;
          line-height: 1.3;
        }
      }
    }
  }

  /* 列表区域 */
  .vg__list {
    display: grid;
    gap: 12px;
    margin: 0;
    padding: 0;
    list-style: none;
    max-height: calc(100vh - 200px);
    overflow-y: auto;
    padding-right: 8px;
    -webkit-overflow-scrolling: touch;
    z-index: 3;

    &::-webkit-scrollbar {
      width: 10px;
    }
    &::-webkit-scrollbar-thumb {
      background: linear-gradient(
        180deg,
        rgba(143, 217, 255, 0.14),
        rgba(95, 195, 255, 0.08)
      );
      border-radius: 8px;
      border: 2px solid transparent;
      background-clip: padding-box;
    }
    &::-webkit-scrollbar-track {
      background: transparent;
    }
  }

  /* 每一项卡片（冷月匣） */
  .vg__item {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    padding: 14px 16px;
    border-radius: 14px;
    background: linear-gradient(
      90deg,
      rgba(6, 12, 18, 0.72),
      rgba(8, 12, 18, 0.78)
    );
    border: 1px solid rgba(95, 195, 255, 0.03);
    backdrop-filter: blur(4px);
    transition: transform 0.15s ease, box-shadow 0.15s ease,
      border-color 0.15s ease, opacity 0.18s ease;
    position: relative;
    overflow: visible;

    &.playing {
      transform: translateY(-4px);
      box-shadow: 0 0 52px rgba(78, 160, 255, 0.12),
        inset 0 2px 12px rgba(143, 217, 255, 0.03);
      border-color: rgba(95, 195, 255, 0.12);
      &::after {
        /* 轻微月环发散 */
        content: "";
        position: absolute;
        right: 12px;
        top: 50%;
        width: 40px;
        height: 40px;
        transform: translateY(-50%);
        border-radius: 50%;
        background: radial-gradient(
          circle,
          rgba(95, 195, 255, 0.06),
          transparent 40%
        );
        filter: blur(6px);
        pointer-events: none;
      }
    }

    &.locked {
      opacity: 0.5;
      filter: grayscale(18%) brightness(0.82);

      .note--locked {
        color: #6a7376;
        font-style: italic;
      }
    }

    .item__left {
      display: flex;
      gap: 12px;
      align-items: center;

      .index {
        min-width: 60px;
        height: 60px;
        border-radius: 12px;
        display: grid;
        place-items: center;
        /* 冷蓝半透明光块替代暖金 */
        background: linear-gradient(180deg, #cfeeff 0%, #9fe8ff 60%);
        color: #04101a;
        font-weight: 800;
        box-shadow: 0 6px 20px rgba(6, 12, 20, 0.36);
        text-shadow: 0 0 6px rgba(0, 0, 0, 0.12);
        border: 1px solid rgba(255, 255, 255, 0.04);
      }

      .info {
        .name {
          color: #eaf7ff;
          font-weight: 700;
          letter-spacing: 0.3px;
        }

        .note {
          color: rgba(223, 248, 255, 0.75);
          font-size: 0.9rem;
          margin-top: 4px;
        }

        .note--locked {
          color: #7a868b;
        }
      }
    }

    .item__right {
      display: flex;
      gap: 10px;
      align-items: center;

      .btn {
        &--icon {
          width: 52px;
          height: 52px;
          border-radius: 12px;
          border: none;
          display: inline-grid;
          place-items: center;
          /* 冷蓝按钮（更“尤诺”） */
          background: linear-gradient(180deg, #6fc9ff, #2f92ff);
          color: #021018;
          font-weight: 700;
          cursor: pointer;
          box-shadow: 0 6px 26px rgba(47, 146, 255, 0.12);
          transition: all 0.15s ease;
          border: 1px solid rgba(255, 255, 255, 0.02);

          &:hover {
            background: linear-gradient(180deg, #8fe6ff, #4ea6ff);
            box-shadow: 0 10px 44px rgba(78, 160, 255, 0.14);
            transform: translateY(-3px);
          }
        }

        &--hint {
          color: rgba(223, 248, 255, 0.6);
        }
      }

      a {
        .el-button {
          background: linear-gradient(180deg, #6fc9ff, #2f92ff);
          border: none;
          color: #021018;
          transition: all 0.15s ease;
          border: 1px solid rgba(255, 255, 255, 0.02);

          &:hover {
            background: linear-gradient(180deg, #8fe6ff, #4ea6ff);
          }
        }
      }
    }
  }

  /* 过渡效果组（背景淡入淡出）*/
  .bg-fade-enter-active,
  .bg-fade-leave-active {
    transition: opacity 900ms ease, transform 900ms ease;
  }
  .bg-fade-enter-from,
  .bg-fade-leave-to {
    opacity: 0;
    transform: scale(1.05);
  }
  .bg-fade-enter-to,
  .bg-fade-leave-from {
    opacity: 1;
    transform: scale(1);
  }

  /* 移动端微调 */
  @media (max-width: 720px) {
    padding: 12px;
    padding-top: 80px;

    .vg__wrap {
      padding: 14px;
    }

    .vg__item {
      padding: 10px;
    }

    .vg__header {
      gap: 8px;
    }

    .index {
      min-width: 48px;
      height: 48px;
      font-size: 0.95rem;
    }
  }
}

/* 轻量动效：月光波动 / 涟漪 */
@keyframes tide-flow {
  0% {
    transform: translateY(0) scale(1);
    opacity: 0.92;
  }
  50% {
    transform: translateY(-6px) scale(1.02);
    opacity: 1;
  }
  100% {
    transform: translateY(0) scale(1);
    opacity: 0.92;
  }
}

@keyframes lunar-ripple {
  0% {
    transform: scale(0.98);
    opacity: 0.18;
  }
  50% {
    transform: scale(1.06);
    opacity: 0.36;
  }
  100% {
    transform: scale(0.98);
    opacity: 0.18;
  }
}
</style>
