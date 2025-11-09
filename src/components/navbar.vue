<template>
  <header class="app-header">
    <h1 class="title">卡提希娅电子设定集</h1>
    <!-- 在线人数展示 -->
    <div class="online-count" v-if="onlineCount !== null">
      当前在线：<span class="count">{{ onlineCount }}人</span>
    </div>
    <!-- 移动端汉堡按钮 -->
    <button
      class="hamburger"
      @click="toggleMobileNav"
      aria-label="Toggle navigation"
    >
      <span :class="{ open: mobileNavOpen }"></span>
      <span :class="{ open: mobileNavOpen }"></span>
      <span :class="{ open: mobileNavOpen }"></span>
    </button>

    <!-- 普通导航 & 移动端下拉导航 -->
    <nav :class="['nav-links', { 'mobile-open': mobileNavOpen }]">
      <RouterLink
        v-for="item in navItems"
        :key="item.name"
        :to="item.path"
        class="nav-item"
        active-class="active-link"
        @click="mobileNavOpen = false"
      >
        {{ item.name }}
      </RouterLink>

      <a
        href="http://36.150.237.25/#/redirector"
        target="_blank"
        rel="noopener"
        class="nav-item"
        active-class="active-link"
        @click="mobileNavOpen = false"
      >
        霜落映界
      </a>
    </nav>
  </header>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from "vue";
import { io } from "socket.io-client";

const navItems = [
  { name: "风旅之始", path: "/" }, // 首页 - 象征流浪骑士的旅程开端
  { name: "时痕刻印", path: "/timeLine" }, // 年谱 - 时间留下的痕迹与记忆
  { name: "剑心低语", path: "/message" }, // 留言板 - 如剑般真诚的内心对话
  { name: "双生瞬影", path: "/gallery" }, // 图集 - 体现卡提希娅与芙露德莉斯双形态
  { name: "圣典残章", path: "/resources" }, // 资料库 - 呼应圣女时期的典籍记载
  { name: "风息回响", path: "/talk" }, // AI对话页面
  { name: "语音馆", path: "/talk" }, // 语音彩蛋页面
  { name: "鸢尾韵律", path: "/music" }, // 歌曲库 - 以她追求的鸢尾花为意象
];

const mobileNavOpen = ref(false);
function toggleMobileNav() {
  mobileNavOpen.value = !mobileNavOpen.value;
}

const siteId = "feibi";

const onlineCount = ref<number | null>(null);

// 连接时带上 query.siteId
const socket: any = io("http://36.150.237.25:3000", {
  query: { siteId },
});

onMounted(() => {
  socket.on("onlineCount", (count: number) => {
    onlineCount.value = count;
  });
});

onBeforeUnmount(() => {
  socket.disconnect();
});
</script>

<style scoped lang="scss">
@import url("https://fonts.googleapis.com/css2?family=Cinzel:ital,wght@0,400;1,700&display=swap");

/* 尤诺风格：月色蓝 / 冷银 / 金边点缀，增加月相、波纹与气流动效 */
.app-header {
  --deep-bg: rgba(6, 10, 28, 0.94); /* 深海夜色 */
  --moon-light: rgba(216, 236, 255, 0.06); /* 月色薄光 */
  --accent: #4ea6ff; /* 主光：月蓝 */
  --accent-2: #8fd9ff; /* 次级渐变：浅天蓝 */
  --silver: #cfeeff; /* 冷银高光 */
  --gold-accent: #ffd78a; /* 金色点缀（饰品/active） */
  --muted-text: #eef6fb; /* 近白带蓝 */
  --aero-sheen: rgba(170, 230, 255, 0.04); /* 气动薄光 */
  --ripple: rgba(78, 160, 255, 0.08); /* 波纹光 */

  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 1000;
  height: 64px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 40px;
  background: radial-gradient(
      420px 90px at 12% 10%,
      rgba(180, 220, 255, 0.02),
      transparent 12%
    ),
    linear-gradient(180deg, rgba(8, 10, 22, 0.98), rgba(6, 8, 18, 0.995));
  backdrop-filter: blur(6px) saturate(1);
  box-shadow: 0 10px 36px rgba(3, 6, 10, 0.62), 0 0 18px var(--moon-light) inset;
  border-bottom: 1px solid rgba(160, 200, 255, 0.03);
  animation: fadeInDown 0.5s ease-out both;
  overflow: visible;

  /* 表面月色薄层与气流 */
  &::after {
    content: "";
    position: absolute;
    left: 0;
    right: 0;
    top: 0;
    height: 100%;
    pointer-events: none;
    background: linear-gradient(180deg, var(--aero-sheen), transparent 28%);
    mix-blend-mode: overlay;
  }

  /* 右侧月相与水纹粒子（替代羽毛） */
  &::before {
    content: "";
    position: absolute;
    right: 6%;
    top: 6%;
    width: 220px;
    height: 84px;
    pointer-events: none;
    background: radial-gradient(
        42px 12px at 18% 22%,
        rgba(140, 200, 255, 0.12),
        transparent 36%
      ),
      radial-gradient(
        20px 8px at 78% 68%,
        rgba(255, 215, 138, 0.04),
        transparent 42%
      ),
      radial-gradient(
        80px 28px at 50% 40%,
        rgba(140, 200, 255, 0.06),
        transparent 46%
      );
    filter: blur(6px);
    transform: translateY(0) rotate(-6deg);
    animation: tide-flow 9s ease-in-out infinite;
    mix-blend-mode: screen;
  }

  .title {
    position: relative;
    font-family: "Cinzel", serif;
    font-style: italic;
    font-size: 26px;
    font-weight: 700;
    color: var(--muted-text);
    background: linear-gradient(90deg, var(--accent), var(--accent-2));
    background-clip: text;
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    letter-spacing: 0.4px;
    text-shadow: 0 6px 18px rgba(4, 8, 14, 0.56);
    transition: transform 0.28s ease, text-shadow 0.28s ease;
    animation: float-slow 10s ease-in-out infinite;

    /* 文字上方月华光 */
    &::before {
      content: "";
      position: absolute;
      left: -6%;
      top: -20%;
      width: 120%;
      height: 140%;
      background: radial-gradient(
        circle at 50% 18%,
        rgba(230, 245, 255, 0.04),
        transparent 20%
      );
      transform: translateY(0);
      animation: shimmer 6.2s linear infinite;
      pointer-events: none;
      mix-blend-mode: screen;
    }

    /* 标题旁的简约新月装饰（呼应月相主题） */
    &::after {
      content: "";
      position: absolute;
      left: -36px;
      top: -6px;
      width: 28px;
      height: 28px;
      border-radius: 50%;
      background: radial-gradient(
          circle at 34% 28%,
          var(--accent-2),
          transparent 52%
        ),
        conic-gradient(
          from 200deg,
          transparent 60%,
          rgba(255, 255, 255, 0.06) 61%
        );
      box-shadow: 0 0 18px rgba(78, 160, 255, 0.1),
        0 1px 0 rgba(255, 215, 138, 0.03) inset;
      transform: rotate(-12deg);
      opacity: 0.98;
      pointer-events: none;
      mask: radial-gradient(circle at 70% 35%, transparent 10%, black 11%);
    }

    &:hover {
      transform: translateY(-2px) scale(1.03);
      text-shadow: 0 10px 36px rgba(78, 160, 255, 0.1);
    }
  }

  .online-count {
    position: relative;
    margin-left: 16px;
    padding: 6px 14px;
    font-family: "Cinzel", serif;
    font-size: 1rem;
    color: var(--muted-text);
    background: linear-gradient(
      135deg,
      rgba(8, 10, 16, 0.28),
      rgba(6, 8, 14, 0.18)
    );
    border: 1px solid rgba(140, 200, 255, 0.04);
    border-radius: 24px;
    backdrop-filter: blur(6px);
    box-shadow: 0 6px 18px rgba(4, 6, 10, 0.44),
      0 0 10px var(--moon-light) inset;
    overflow: hidden;
    cursor: default;
    transition: transform 0.22s ease, box-shadow 0.22s ease;

    &:hover {
      transform: translateY(-2px);
      box-shadow: 0 14px 36px rgba(3, 4, 6, 0.56),
        0 0 34px rgba(140, 200, 255, 0.06);
    }

    .count {
      display: inline-block;
      margin-left: 18px;
      font-weight: 700;
      color: var(--accent-2);
      background: linear-gradient(90deg, var(--accent), var(--accent-2));
      -webkit-background-clip: text;
      -webkit-text-fill-color: transparent;
      text-shadow: 0 0 8px rgba(100, 190, 255, 0.06);
    }
  }

  .nav-links {
    display: flex;
    gap: 22px;
    align-items: center;

    .nav-item {
      position: relative;
      color: var(--muted-text);
      font-weight: 500;
      text-decoration: none;
      transition: color 0.22s ease, transform 0.16s ease;
      padding: 6px 4px;
      border-radius: 6px;
      overflow: visible;
      font-family: "STKaiti", "KaiTi", "Noto Serif SC", serif;
      font-style: italic;

      &::after {
        content: "";
        position: absolute;
        left: 50%;
        bottom: -8px;
        width: 0;
        height: 6px;
        border-radius: 6px;
        background: linear-gradient(
          90deg,
          rgba(0, 0, 0, 0),
          rgba(78, 160, 255, 0.92),
          rgba(143, 217, 255, 0.9),
          rgba(0, 0, 0, 0)
        );
        transform: translateX(-50%);
        opacity: 0.95;
        filter: blur(0.8px) contrast(1.03);
        transition: width 0.36s cubic-bezier(0.2, 0.9, 0.2, 1), left 0.36s,
          opacity 0.24s;
        background-size: 200% 100%;
        animation: flow-wave 6.8s linear infinite;
      }

      /* 替换为小月点与气流晕圈 */
      &::before {
        content: "";
        position: absolute;
        right: 14%;
        top: -6px;
        width: 10px;
        height: 10px;
        border-radius: 50%;
        background: radial-gradient(circle, var(--accent-2), transparent 60%);
        opacity: 0;
        transform: translateY(-6px) scale(0.86);
        transition: opacity 0.26s, transform 0.36s;
        box-shadow: 0 6px 14px rgba(100, 190, 255, 0.08);
      }

      &:hover {
        color: var(--accent-2);
        transform: translateY(-1.8px);
        text-shadow: 0 0 8px rgba(100, 190, 255, 0.04);
      }

      &:hover::after {
        width: 72%;
        left: 50%;
        opacity: 1;
      }

      &:hover::before {
        opacity: 1;
        transform: translateY(0) scale(1);
      }
    }

    .active-link {
      color: var(--accent);
      font-weight: 600;

      /* 活动项用金色小月与微光（代表祝福/圣仪） */
      &::before {
        content: "☾";
        position: absolute;
        right: -14px;
        top: 50%;
        transform: translateY(-50%) rotate(-8deg);
        font-size: 12px;
        color: var(--gold-accent);
        text-shadow: 0 2px 10px rgba(255, 215, 138, 0.12);
        animation: lunar-glow 3.6s ease-in-out infinite;
        opacity: 0.98;
      }

      &::after {
        width: 92%;
        opacity: 1;
        box-shadow: 0 6px 22px rgba(100, 190, 255, 0.06);
      }
    }
  }

  .hamburger {
    display: none;
    flex-direction: column;
    justify-content: space-around;
    width: 28px;
    height: 24px;
    background: none;
    border: none;
    cursor: pointer;
    padding: 0;

    span {
      display: block;
      width: 100%;
      height: 3px;
      background: rgba(238, 246, 251, 0.92);
      border-radius: 2px;
      transition: transform 0.28s ease, opacity 0.28s ease, background 0.28s;
      box-shadow: 0 2px 6px rgba(8, 6, 10, 0.18), 0 0 8px var(--moon-light);
    }

    span.open:nth-child(1) {
      transform: translateY(10px) rotate(45deg);
      background: linear-gradient(90deg, var(--accent), var(--accent-2));
    }

    span.open:nth-child(2) {
      opacity: 0;
    }

    span.open:nth-child(3) {
      transform: translateY(-10px) rotate(-45deg);
      background: linear-gradient(90deg, var(--accent), var(--accent-2));
    }
  }

  @media (max-width: 768px) {
    padding: 0 20px;

    .title {
      display: none;
    }
    .hamburger {
      display: flex;
    }

    .nav-links {
      position: absolute;
      top: 64px;
      left: 0;
      right: 0;
      flex-direction: column;
      background: linear-gradient(
        180deg,
        rgba(8, 6, 12, 0.98),
        rgba(6, 4, 10, 0.995)
      );
      backdrop-filter: blur(12px);
      gap: 0;
      overflow: hidden;
      max-height: 0;
      transition: max-height 0.34s ease;
      border-top: 1px solid rgba(180, 220, 255, 0.03);

      &.mobile-open {
        max-height: 520px;
      }

      .nav-item {
        padding: 14px 20px;
        border-bottom: 1px solid rgba(255, 255, 255, 0.03);
      }
    }
  }
}

/* 动效关键帧（改为月相/气流/波纹节奏） */
@keyframes flow-wave {
  0% {
    background-position: 0% 50%;
  }
  50% {
    background-position: 100% 50%;
  }
  100% {
    background-position: 0% 50%;
  }
}

@keyframes float-slow {
  0% {
    transform: translateY(0);
  }
  50% {
    transform: translateY(-4px);
  }
  100% {
    transform: translateY(0);
  }
}

@keyframes fadeInDown {
  0% {
    opacity: 0;
    transform: translateY(-8px);
  }
  100% {
    opacity: 1;
    transform: translateY(0);
  }
}

/* 由原来的 feather-drift 改为 tide-flow（更像气流与月潮） */
@keyframes tide-flow {
  0% {
    transform: translateY(0) rotate(-6deg) translateX(0);
    opacity: 0.85;
    filter: blur(6px) saturate(1);
  }
  50% {
    transform: translateY(-8px) rotate(2deg) translateX(-6px);
    opacity: 1;
    filter: blur(4px) saturate(1.02);
  }
  100% {
    transform: translateY(0) rotate(-6deg) translateX(0);
    opacity: 0.85;
  }
}

@keyframes shimmer {
  0% {
    transform: translateY(0) rotate(0deg);
    opacity: 0.6;
  }
  50% {
    transform: translateY(-6px) rotate(0.4deg);
    opacity: 1;
  }
  100% {
    transform: translateY(0) rotate(0deg);
    opacity: 0.6;
  }
}

@keyframes lunar-glow {
  0% {
    transform: translateY(-6%) rotate(-6deg);
    opacity: 0.8;
    filter: drop-shadow(0 2px 8px rgba(78, 160, 255, 0.06));
  }
  50% {
    transform: translateY(6%) rotate(2deg);
    opacity: 1;
    filter: drop-shadow(0 4px 18px rgba(140, 200, 255, 0.12));
  }
  100% {
    transform: translateY(-6%) rotate(-6deg);
    opacity: 0.8;
  }
}
</style>
