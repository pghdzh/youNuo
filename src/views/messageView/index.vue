<template>
  <div class="megumi-message-board" aria-live="polite">
    <!-- 背景轮播（两组用于桌面/移动不同裁切） -->
    <div class="carousel carousel1" aria-hidden="true">
      <img
        v-for="(src, idx) in randomFive"
        :key="idx"
        :src="src"
        class="carousel-image"
        :class="{ active: idx === currentIndex }"
      />
    </div>
    <div class="carousel carousel2" aria-hidden="true">
      <img
        v-for="(src, idx) in randomFive2"
        :key="idx"
        :src="src"
        class="carousel-image"
        :class="{ active: idx === currentIndex }"
      />
    </div>
    <!-- 半透明顶部标题 -->
    <header class="board-header" role="banner">
      <div class="title-wrap">
        <h1>留言板</h1>
        <span class="title-count">（共{{ count }}条）</span>

        <p class="subtitle">剑铭心迹，风聆汝音</p>
      </div>
    </header>

    <!-- 留言展示区 -->
    <section class="message-list">
      <transition-group name="msg" tag="div" class="message-list-inner">
        <div v-if="loading" class="skeleton-wrap" key="skeleton">
          <div class="skeleton" v-for="i in 1" :key="i">
            <div class="sk-avatar"></div>
            <div class="sk-lines">
              <div class="sk-line short"></div>
              <div class="sk-line"></div>
            </div>
          </div>
        </div>
        <div
          v-for="(msg, idx) in messages"
          :key="msg.id || msg._tempId || idx"
          class="message-card"
          :data-index="idx"
          tabindex="0"
          role="article"
          :aria-label="`留言来自 ${msg.name || '匿名'}，内容：${msg.content}`"
        >
          <div class="message-meta">
            <div class="left-meta">
              <div class="name-avatar" :title="msg.name || '匿名'">
                {{ getInitial(msg.name) }}
              </div>
              <div class="meta-texts">
                <div class="message-name">{{ msg.name || "匿名" }}</div>
                <div class="message-time">{{ formatTime(msg.created_at) }}</div>
              </div>
            </div>
            <div class="shouan-icon" role="button" tabindex="0">
              <svg
                xmlns="http://www.w3.org/2000/svg"
                viewBox="0 0 128 128"
                role="img"
                aria-labelledby="title desc"
              >
                <title id="title">尤诺风格·新月图标</title>
                <desc id="desc">
                  简约的新月图标，冷蓝与银色渐变，辅以纤细光环与微小星点
                </desc>

                <defs>
                  <!-- 背景：深夜渐变方块 -->
                  <linearGradient id="bgG" x1="0" x2="1" y1="0" y2="1">
                    <stop offset="0%" stop-color="#061426" />
                    <stop offset="100%" stop-color="#020612" />
                  </linearGradient>

                  <!-- 新月主色：冷蓝到银白 -->
                  <linearGradient id="moonG" x1="0" x2="1" y1="0" y2="1">
                    <stop offset="0%" stop-color="#5fc3ff" />
                    <stop offset="65%" stop-color="#dff8ff" />
                    <stop offset="100%" stop-color="#f8feff" />
                  </linearGradient>

                  <!-- 纤细环渐变（冷蓝→金光细节点缀） -->
                  <linearGradient id="ringG" x1="0" x2="1">
                    <stop offset="0%" stop-color="#8fd9ff" />
                    <stop offset="70%" stop-color="#cfeeff" />
                    <stop offset="100%" stop-color="#ffd98a" />
                  </linearGradient>

                  <!-- 轻微投影 -->
                  <filter id="ds" x="-50%" y="-50%" width="200%" height="200%">
                    <feDropShadow
                      dx="0"
                      dy="6"
                      stdDeviation="8"
                      flood-color="#000"
                      flood-opacity="0.5"
                    />
                  </filter>

                  <!-- 小星形（复用）-->
                  <symbol id="star" viewBox="0 0 10 10">
                    <path
                      d="M5 0.8 L6.1 3.1 L8.7 3.3 L6.3 4.9 L7 7.6 L5 6.2 L3 7.6 L3.7 4.9 L1.3 3.3 L3.9 3.1 Z"
                      fill="currentColor"
                    />
                  </symbol>

                  <!-- 用于做新月的遮罩（左半为实，右半被切掉） -->
                  <mask id="crescentMask">
                    <!-- 全白表示可见 -->
                    <rect x="0" y="0" width="128" height="128" fill="white" />
                    <!-- 右侧较大的黑色圆形遮去部分形成新月空隙 -->
                    <circle cx="78" cy="52" r="28" fill="black" />
                  </mask>
                </defs>

                <!-- 背景圆角块 -->
                <rect
                  width="128"
                  height="128"
                  rx="18"
                  ry="18"
                  fill="url(#bgG)"
                />

                <!-- 软光环（顶部偏移，增加层次） -->
                <ellipse
                  cx="64"
                  cy="18"
                  rx="36"
                  ry="9"
                  fill="rgba(140,200,255,0.06)"
                />

                <!-- 新月主体：使用渐变并套用遮罩 -->
                <g filter="url(#ds)">
                  <circle
                    cx="56"
                    cy="56"
                    r="30"
                    fill="url(#moonG)"
                    mask="url(#crescentMask)"
                  />
                </g>

                <!-- 细线光环：半环（用弧 path）-->
                <path
                  d="M32 40 A40 40 0 0 1 96 44"
                  fill="none"
                  stroke="url(#ringG)"
                  stroke-width="2.2"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  opacity="0.95"
                />

                <!-- 更内侧的微细环（微偏移，制造层次） -->
                <path
                  d="M36 44 A34 34 0 0 1 92 48"
                  fill="none"
                  stroke="url(#ringG)"
                  stroke-width="1.0"
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  opacity="0.65"
                />

                <!-- 三个点缀星（不同大小与透明度） -->
                <g fill="#fff8e8" opacity="0.95" transform="translate(0,-2)">
                  <use
                    href="#star"
                    x="22"
                    y="22"
                    width="6"
                    height="6"
                    style="color: #dff8ff; opacity: 0.2"
                  />
                  <use
                    href="#star"
                    x="94"
                    y="20"
                    width="5"
                    height="5"
                    style="color: #ffd98a; opacity: 0.22"
                  />
                  <use
                    href="#star"
                    x="78"
                    y="70"
                    width="4"
                    height="4"
                    style="color: #e8fbff; opacity: 0.18"
                  />
                </g>

                <!-- 底部轻投影（平衡构图） -->
                <ellipse
                  cx="64"
                  cy="116"
                  rx="28"
                  ry="4.0"
                  fill="rgba(0,0,0,0.28)"
                />
              </svg>
            </div>
          </div>

          <p class="message-content">{{ msg.content }}</p>
        </div>
      </transition-group>
    </section>

    <!-- 底部发送区 -->
    <section class="message-form" aria-label="写下你的留言">
      <label class="sr-only" for="mb-name">你的昵称</label>
      <input
        id="mb-name"
        v-model="name"
        type="text"
        placeholder="你的昵称"
        @keydown.enter.prevent
      />

      <label class="sr-only" for="mb-content">留言内容</label>
      <textarea
        id="mb-content"
        v-model="content"
        placeholder="写下你的留言..."
        @keydown.ctrl.enter.prevent="submitMessage"
        @input="autoGrow"
        ref="textareaRef"
      />

      <div class="form-row">
        <div class="hint">按 <kbd>Ctrl</kbd> + <kbd>Enter</kbd> 快捷发送</div>
        <button @click="submitMessage" :disabled="isSending || !content.trim()">
          <span v-if="!isSending">发送</span>
          <span v-else>发送中…</span>
        </button>
      </div>
    </section>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, nextTick, onBeforeUnmount } from "vue";
import { getMessageList, createMessage } from "@/api/modules/message";

const messages = ref<any[]>([]);
const count = ref(0);
const name = ref(localStorage.getItem("message_name") || "");
const content = ref("");
const loading = ref(true);
const isSending = ref(false);

const textareaRef = ref<HTMLTextAreaElement | null>(null);

const fetchMessages = async () => {
  loading.value = true;
  try {
    const res = await getMessageList({ page: 1, pageSize: 9999 });
    messages.value = res.data || [];
    count.value = res.pagination.total;
    await nextTick();
  } catch (err) {
    console.error(err);
  } finally {
    loading.value = false;
  }
};

const submitMessage = async () => {
  if (!content.value.trim() || isSending.value) return;
  isSending.value = true;
  const payload = { name: name.value || "匿名", content: content.value };
  try {
    localStorage.setItem("message_name", name.value);
    content.value = "";
    await nextTick();
    // 发送请求
    await createMessage(payload);
    // 重新同步列表（更可靠）
    await fetchMessages();
  } catch (err) {
    console.error(err);
  } finally {
    isSending.value = false;
  }
};

const formatTime = (time: string) => {
  if (!time) return "";
  const d = new Date(time);
  // 例如：2025-08-11 15:30
  const y = d.getFullYear();
  const m = String(d.getMonth() + 1).padStart(2, "0");
  const day = String(d.getDate()).padStart(2, "0");
  const hh = String(d.getHours()).padStart(2, "0");
  const mm = String(d.getMinutes()).padStart(2, "0");
  return `${y}-${m}-${day} ${hh}:${mm}`;
};

const getInitial = (n?: string) => {
  if (!n) return "匿";
  return n.trim().slice(0, 1).toUpperCase();
};

const autoGrow = (e?: Event) => {
  const ta = textareaRef.value;
  if (!ta) return;
  ta.style.height = "auto";
  const h = Math.min(ta.scrollHeight, 220);
  ta.style.height = h + "px";
};

// ========== 背景图片导入与轮播 ==========
const modules = import.meta.glob("@/assets/images1/*.{jpg,png,jpeg,webp}", {
  eager: true,
});
const allSrcs: string[] = Object.values(modules).map((mod: any) => mod.default);

const modules2 = import.meta.glob("@/assets/images2/*.{jpg,png,jpeg,webp}", {
  eager: true,
});
const allSrcs2: string[] = Object.values(modules2).map(
  (mod: any) => mod.default
);

function shuffle<T>(arr: T[]): T[] {
  const a = arr.slice();
  for (let i = a.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [a[i], a[j]] = [a[j], a[i]];
  }
  return a;
}
const randomFive = ref<string[]>(shuffle(allSrcs).slice(0, 5));
const randomFive2 = ref<string[]>(shuffle(allSrcs2).slice(0, 5));

const currentIndex = ref(0);
let Imgtimer: number | undefined;

onMounted(() => {
  fetchMessages();
  Imgtimer = window.setInterval(() => {
    currentIndex.value =
      (currentIndex.value + 1) % Math.max(1, randomFive.value.length);
  }, 5200);
  nextTick(() => autoGrow());
});

onBeforeUnmount(() => {
  if (Imgtimer) clearInterval(Imgtimer);
});
</script>

<style scoped lang="scss">
/* 卡提希娅色板（变量名保持不变） */
$bg-start: #050714; // 深海夜色（更冷的湮灭底）
$bg-end: #071028; // 次深底（深紫-海蓝过渡）

$accent-1: #4ea6ff;
$accent-2: #8fd9ff;
$text-main: #eae9ee;
$text-muted: rgba(234, 233, 238, 0.78);

$card-bg: rgba(6, 8, 12, 0.46); // 暗紫玻璃卡片（含湿感）
$card-border: rgba(95, 224, 255, 0.04); // 卡片边框走海蓝高光

$soft-shadow: rgba(0, 0, 0, 0.64);
$inner-glow: rgba(95, 224, 255, 0.02); // 内发光：偏海蓝的微弱光晕

$blood-halo: rgba(255, 122, 163, 0.04); // 毒色/点缀（保留少量粉红-毒感）

.megumi-message-board {
  position: relative;
  min-height: 100vh;
  padding-top: 110px;
  display: flex;
  flex-direction: column;
  background: linear-gradient(180deg, $bg-start 0%, $bg-end 100%);
  font-family: "Noto Sans SC", "Noto Sans", system-ui, -apple-system, "Segoe UI",
    Roboto, Arial;
  color: $text-main;
  overflow: hidden;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;

  /* 整页弱雾与暗紫颗粒（可做背景氛围） */
  &::after {
    content: "";
    position: absolute;
    inset: 0;
    z-index: 0;
    pointer-events: none;
    background-image: radial-gradient(
        circle at 12% 18%,
        rgba($accent-1, 0.04) 0 2px,
        transparent 6px
      ),
      radial-gradient(
        circle at 70% 40%,
        rgba($accent-2, 0.03) 0 2px,
        transparent 6px
      );
    mix-blend-mode: screen;
    filter: blur(6px);
    opacity: 0.95;
  }

  .carousel {
    position: absolute;
    inset: 0;
    z-index: 0;
    pointer-events: none;

    &::before {
      content: "";
      position: absolute;
      inset: 0;
      background: linear-gradient(
        180deg,
        rgba(4, 6, 8, 0.16),
        rgba(6, 4, 6, 0.32)
      );
      z-index: 1;
      mix-blend-mode: soft-light;
    }

    .carousel-image {
      position: absolute;
      width: 100%;
      height: 100%;
      object-fit: cover;
      opacity: 0;
      transition: opacity 1s ease, transform 10s linear;
      filter: blur(0.6px) saturate(0.72) contrast(0.95) brightness(0.88);
      transform: scale(1.05);

      &.active {
        opacity: 1;
        transform: scale(1);
      }
    }
  }

  .carousel2 {
    display: none;
  }

  /* ---------- 顶部标题（改为乐谱 + 暗紫珍珠玻璃） ---------- */
  .board-header {
    position: absolute;
    top: 72px;
    left: 50%;
    transform: translateX(-50%);
    width: calc(100% - 32px);
    max-width: 960px;
    padding: 14px 18px;
    border-radius: 14px;
    box-shadow: 0 18px 48px rgba(0, 0, 0, 0.7), inset 0 1px 0 $inner-glow;
    backdrop-filter: blur(8px) saturate(0.9);
    z-index: 6;
    border: 1px solid rgba(255, 107, 133, 0.02);
    background: linear-gradient(
      180deg,
      rgba(10, 8, 10, 0.16),
      rgba(6, 6, 8, 0.22)
    );

    /* 乐谱细线（非常低不透明） */
    &::before {
      content: "";
      position: absolute;
      inset: 6% 0 6% 0;
      z-index: -1;
      pointer-events: none;
      background-image: repeating-linear-gradient(
        to bottom,
        rgba(255, 255, 255, 0.02) 0px,
        rgba(255, 255, 255, 0.02) 1px,
        transparent 1px,
        transparent 18px
      );
      opacity: 0.06;
      mix-blend-mode: overlay;
      transform: translateY(-8%);
      animation: staff-scroll 20s linear infinite;
    }

    .title-wrap {
      display: flex;
      align-items: center;
      gap: 12px;

      h1 {
        margin: 0;
        font-size: 18px;
        color: $accent-2;
        letter-spacing: 0.4px;
        font-weight: 900;
        text-shadow: 0 4px 18px rgba(0, 0, 0, 0.6);
        position: relative;
      }

      .title-count {
        color: rgba($text-main, 0.92);
        font-size: 12px;
        font-weight: 700;
        margin-left: 6px;
      }

      .subtitle {
        margin: 0;
        margin-left: auto;
        color: rgba($text-main, 0.94);
        font-size: 13px;
        font-weight: 600;
        text-shadow: 0 1px 6px rgba($accent-1, 0.02);
      }
    }
  }

  /* ---------- 留言列表 ---------- */
  .message-list {
    z-index: 2;
    position: relative;
    flex: 1;
    overflow-y: auto;
    padding: 28px 20px 340px;
    margin-top: 18px;

    .message-list-inner {
      max-width: 960px;
      max-height: 80vh;
      margin: 0 auto;
      display: flex;
      flex-direction: column;
      gap: 16px;
      position: relative;
      z-index: 2;
      overflow-y: auto;
    }

    .skeleton-wrap {
      display: flex;
      flex-direction: column;
      gap: 12px;

      .skeleton {
        display: flex;
        gap: 12px;
        align-items: center;
        padding: 12px;
        background: linear-gradient(
          180deg,
          rgba(8, 6, 8, 0.28),
          rgba(10, 6, 8, 0.22)
        );
        border-radius: 12px;
        box-shadow: 0 6px 18px rgba(0, 0, 0, 0.6);
        border: 1px solid rgba($accent-1, 0.02);
      }

      .sk-avatar {
        width: 44px;
        height: 44px;
        border-radius: 10px;
        background: linear-gradient(90deg, $accent-1, $accent-2);
      }

      .sk-lines {
        flex: 1;

        .sk-line {
          height: 10px;
          border-radius: 6px;
          background: linear-gradient(
            90deg,
            rgba(255, 255, 255, 0.03),
            rgba(200, 200, 200, 0.02)
          );
          margin-bottom: 8px;
        }

        .sk-line.short {
          width: 40%;
        }
      }
    }
  }

  /* ---------- 单条消息卡片（卡提希娅风） ---------- */
  .message-card {
    background: linear-gradient(
      180deg,
      rgba(6, 6, 8, 0.36),
      rgba(8, 6, 10, 0.48)
    );
    border-radius: 14px;
    padding: 14px 16px;
    margin: 6px auto;
    width: calc(100% - 48px);
    max-width: 960px;
    border: 1px solid $card-border;
    transition: transform 0.32s cubic-bezier(0.2, 0.9, 0.3, 1), box-shadow 0.32s,
      border-color 0.32s;
    transform-origin: center;
    position: relative;
    z-index: 3;
    overflow: visible;
    box-shadow: 0 14px 44px $soft-shadow, inset 0 1px 0 $inner-glow;

    &:hover {
      transform: translateY(-8px) scale(1.01);
      box-shadow: 0 32px 96px rgba(0, 0, 0, 0.75);
      border-color: rgba($accent-2, 0.08);
      background: linear-gradient(
        180deg,
        rgba(8, 6, 10, 0.52),
        rgba(6, 4, 8, 0.6)
      );
    }

    .message-meta {
      display: flex;
      justify-content: space-between;
      align-items: flex-start;
      gap: 12px;
      margin-bottom: 8px;

      .left-meta {
        display: flex;
        gap: 12px;
        align-items: center;

        .name-avatar {
          width: 48px;
          height: 48px;
          border-radius: 10px;
          display: inline-flex;
          align-items: center;
          justify-content: center;
          font-weight: 900;
          color: #0a0710;
          background: linear-gradient(180deg, $accent-2 0%, $accent-1 60%);

          box-shadow: inset 0 -6px 18px rgba(0, 0, 0, 0.35);
          font-size: 16px;
          flex-shrink: 0;
          position: relative;

          /* 头像旁的微血光（左下） */
          &::after {
            content: "";
            position: absolute;
            left: -6px;
            bottom: -6px;
            width: 10px;
            height: 10px;
            border-radius: 50%;
            background: radial-gradient(
              circle,
              rgba($accent-2, 0.95) 0%,
              rgba($accent-1, 0.7) 40%,
              transparent 70%
            );
            filter: blur(2px);
            opacity: 0.92;
          }
        }

        .meta-texts {
          .message-name {
            font-size: 15px;
            color: $accent-2;
            font-weight: 800;
            line-height: 1;
            text-shadow: 0 4px 14px rgba(0, 0, 0, 0.5);
          }

          .message-time {
            font-size: 12px;
            color: rgba($text-main, 0.78);
            margin-top: 2px;
          }
        }
      } /* left-meta end */

      .shouan-icon {
        display: inline-grid;
        place-items: center;
        width: 52px;
        height: 52px;
        border-radius: 12px;
        cursor: pointer;
        user-select: none;
        position: relative;
        z-index: 4;
        box-shadow: 0 10px 36px rgba(0, 0, 0, 0.6), inset 0 1px 0 $inner-glow;
        transition: transform 260ms cubic-bezier(0.2, 0.9, 0.3, 1),
          box-shadow 260ms, background 260ms;
        will-change: transform, box-shadow, opacity;
        animation: iconFloat 8s ease-in-out infinite;

        svg {
          width: 54px;
          height: 54px;
          display: block;
          overflow: visible;
        }
      }

      @keyframes iconFloat {
        0% {
          transform: translateY(0);
        }
        40% {
          transform: translateY(-6px);
        }
        70% {
          transform: translateY(-3px);
        }
        100% {
          transform: translateY(0);
        }
      }
    } /* message-meta end */

    .message-content {
      font-size: 15px;
      color: rgba($text-main, 0.96);
      line-height: 1.7;
      white-space: pre-wrap;
      word-break: break-word;
      margin: 0;
      padding-bottom: 2px;
      letter-spacing: 0.2px;
    }
  } /* .message-card end */

  /* ---------- 固定底部输入区（暗紫血色按钮） ---------- */
  .message-form {
    position: fixed;
    left: 50%;
    transform: translateX(-50%);
    bottom: 18px;
    width: calc(100% - 32px);
    max-width: 960px;
    background: linear-gradient(
      180deg,
      rgba(8, 6, 8, 0.92),
      rgba(6, 4, 6, 0.98)
    );
    padding: 14px;
    border-radius: 14px;
    display: flex;
    flex-direction: column;
    gap: 10px;
    box-shadow: 0 28px 80px rgba(0, 0, 0, 0.85), inset 0 1px 0 $inner-glow;
    z-index: 6;
    border: 1px solid rgba($accent-2, 0.02);
    will-change: transform, opacity;

    input,
    textarea {
      padding: 12px 14px;
      border-radius: 12px;
      border: 1px solid rgba($accent-2, 0.03);
      font-size: 14px;
      outline: none;
      transition: box-shadow 0.18s, border-color 0.18s, background 0.18s,
        transform 0.12s;
      background: linear-gradient(
        180deg,
        rgba(10, 10, 10, 0.7),
        rgba(8, 6, 8, 0.78)
      );
      box-shadow: inset 0 -4px 10px rgba(0, 0, 0, 0.48);
      color: $text-main;
      resize: none;
      -webkit-appearance: none;
      -moz-appearance: none;
    }

    input::placeholder,
    textarea::placeholder {
      color: rgba($text-main, 0.42);
    }

    input:focus,
    textarea:focus {
      border-color: $accent-1;
      box-shadow: 0 14px 42px rgba($accent-1, 0.06),
        inset 0 -6px 12px rgba(0, 0, 0, 0.36);
      background: linear-gradient(
        180deg,
        rgba(12, 8, 10, 0.76),
        rgba(10, 6, 8, 0.78)
      );
      transform: translateY(-1px);
    }

    textarea {
      min-height: 64px;
      max-height: 220px;
      line-height: 1.6;
    }

    .form-row {
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: 12px;

      .hint {
        color: rgba($text-main, 0.86);
        font-size: 13px;
        display: flex;
        align-items: center;
        gap: 8px;

        kbd {
          background: rgba(8, 6, 8, 0.78);
          border-radius: 6px;
          padding: 3px 7px;
          border: 1px solid rgba($accent-2, 0.02);
          font-size: 12px;
          box-shadow: inset 0 -2px 6px rgba(0, 0, 0, 0.28);
          color: $accent-2;
          font-weight: 700;
          letter-spacing: 0.6px;
        }
      }

      button {
        padding: 10px 18px;
        background: linear-gradient(180deg, $accent-1 0%, $accent-2 60%);
        color: #0a0710;
        border: none;
        border-radius: 12px;
        cursor: pointer;
        font-weight: 800;
        box-shadow: 0 14px 48px rgba($accent-2, 0.06),
          inset 0 1px 0 rgba(255, 255, 255, 0.02);
        transition: transform 0.14s ease, box-shadow 0.14s ease,
          opacity 0.14s ease;
        display: inline-flex;
        align-items: center;
        gap: 8px;

        &::after {
          content: "";
          display: inline-block;
          width: 6px;
          height: 6px;
          border-radius: 50%;
          background: radial-gradient(
            circle at 40% 40%,
            $accent-2 0%,
            $accent-1 40%,
            transparent 60%
          );
          box-shadow: 0 6px 18px rgba($accent-2, 0.06);
        }

        &:hover {
          transform: translateY(-3px) scale(1.02);
          box-shadow: 0 22px 64px rgba($accent-2, 0.09);
        }
        &:active {
          transform: translateY(-1px) scale(0.995);
        }
      }

      button:disabled {
        opacity: 0.52;
        cursor: not-allowed;
        transform: none;
        box-shadow: none;
        background: linear-gradient(
          180deg,
          rgba(10, 10, 10, 0.6),
          rgba(8, 8, 8, 0.6)
        );
        color: rgba(220, 240, 235, 0.6);
      }
    }

    &.pulse {
      animation: formPulse 1200ms ease-in-out 1;
    }
  }

  @keyframes formPulse {
    0% {
      transform: translateX(-50%) scale(1);
      box-shadow: 0 28px 80px rgba(0, 0, 0, 0.85);
    }
    40% {
      transform: translateX(-50%) scale(1.01);
      box-shadow: 0 36px 96px rgba($accent-2, 0.08);
    }
    100% {
      transform: translateX(-50%) scale(1);
      box-shadow: 0 28px 80px rgba(0, 0, 0, 0.85);
    }
  }

  /* 浮动音符（模板内可渲染若干 .floating-note） */
  .floating-note {
    position: absolute;
    font-size: 14px;
    color: $accent-2;
    opacity: 0.95;
    transform-origin: center;
    animation: note-float 4.6s ease-in-out infinite;
    filter: drop-shadow(0 8px 20px rgba($accent-2, 0.06));
  }

  @keyframes staff-scroll {
    0% {
      transform: translateY(-6%);
    }
    50% {
      transform: translateY(6%);
    }
    100% {
      transform: translateY(-6%);
    }
  }
  @keyframes note-float {
    0% {
      transform: translateY(0) rotate(-6deg) scale(0.95);
      opacity: 0.85;
    }
    50% {
      transform: translateY(-12px) rotate(3deg) scale(1);
      opacity: 1;
    }
    100% {
      transform: translateY(0) rotate(-6deg) scale(0.95);
      opacity: 0.85;
    }
  }

  /* ---------- 响应式：移动端收敛（保留你的逻辑） ---------- */
  @media (max-width: 980px) {
    padding-top: 90px;

    .carousel1 {
      display: none;
    }
    .carousel2 {
      display: block;
    }

    .board-header {
      left: 12px;
      transform: none;
      width: calc(100% - 24px);
    }

    .message-list {
      padding: 18px 12px 260px;
      .message-list-inner {
        gap: 12px;
      }
    }

    .message-card {
      width: calc(100% - 28px);
      border-radius: 12px;
      padding: 12px;
      .name-avatar {
        width: 44px;
        height: 44px;
      }
    }

    .message-form {
      left: 12px;
      transform: none;
      width: calc(100% - 24px);
      bottom: 12px;
      padding: 12px;
    }
  }

  /* 无障碍隐藏 */
  .sr-only {
    position: absolute !important;
    width: 1px !important;
    height: 1px !important;
    padding: 0 !important;
    margin: -1px !important;
    overflow: hidden !important;
    clip: rect(0, 0, 0, 0) !important;
    white-space: nowrap !important;
    border: 0 !important;
  }
}
</style>
