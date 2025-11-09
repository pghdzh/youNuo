<template>
  <div class="chat-page">
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

    <div class="chat-container">
      <!-- 统计面板（放在聊天容器顶部） -->
      <div class="stats-panel">
        <div class="stat-item">
          总对话次数：<span>{{ stats.totalChats }}</span>
        </div>
        <div class="stat-item">
          首次使用：<span>{{
            new Date(stats.firstTimestamp).toISOString().slice(0, 10)
          }}</span>
        </div>
        <div class="stat-item">
          活跃天数：<span>{{ stats.activeDates.length }}</span> 天
        </div>
        <div class="stat-item">
          今日对话：<span>{{
            stats.dailyChats[new Date().toISOString().slice(0, 10)] || 0
          }}</span>
          次
        </div>
        <button class="detail-btn" @click="showModal = true">全部</button>
      </div>
      <div class="messages" ref="msgList">
        <transition-group name="msg" tag="div">
          <div
            v-for="msg in chatLog"
            :key="msg.id"
            :class="[
              'message',
              msg.role,
              { error: msg.isError, egg: msg.isEgg },
            ]"
          >
            <div class="avatar" :class="msg.role"></div>
            <div class="bubble">
              <div class="content" v-html="msg.text"></div>
            </div>
          </div>
          <div v-if="loading" class="message bot" key="loading">
            <div class="avatar bot"></div>
            <div class="bubble loading">
              正在思考中
              <span class="dots">
                <span class="dot">.</span>
                <span class="dot">.</span>
                <span class="dot">.</span>
              </span>
            </div>
          </div>
        </transition-group>
      </div>
      <form class="input-area" @submit.prevent="sendMessage">
        <!-- 输入框改成 textarea -->
        <textarea
          v-model="input"
          placeholder="向长离提问…"
          :disabled="loading"
          @keydown="handleKeydown"
          rows="1"
        ></textarea>

        <!-- 清空按钮 -->
        <div class="btn-group">
          <button
            type="button"
            class="clear-btn"
            @click="clearChat"
            :disabled="loading"
            title="清空对话"
          >
            ✖
          </button>
        </div>

        <!-- 发送按钮 -->
        <button
          type="submit"
          class="send-btn"
          :disabled="!input.trim() || loading"
        >
          发送
        </button>

        <!-- 统计数据按钮 -->
        <button
          type="button"
          class="Alldetail-btn"
          @click="showModal = true"
          title="查看统计"
        >
          统计数据
        </button>
      </form>
    </div>

    <!-- 详细统计弹窗 -->
    <div v-if="showModal" class="modal-overlay" @click.self="showModal = false">
      <div class="modal-content">
        <h3>详细统计</h3>
        <ul class="detail-list">
          <li>总对话次数：{{ stats.totalChats }}</li>
          <li>
            首次使用：{{
              new Date(stats.firstTimestamp).toISOString().slice(0, 10)
            }}
          </li>
          <li>活跃天数：{{ stats.activeDates.length }} 天</li>
          <li>
            今日对话：{{
              stats.dailyChats[new Date().toISOString().slice(0, 10)] || 0
            }}
            次
          </li>
          <li>总使用时长：{{ formatDuration(stats.totalTime) }}</li>
          <li>当前连续活跃：{{ stats.currentStreak }} 天</li>
          <li>最长连续活跃：{{ stats.longestStreak }} 天</li>
          <li>
            最活跃日：{{ mostActiveDayComputed }} （{{
              stats.dailyChats[mostActiveDayComputed] || 0
            }}
            次）
          </li>
        </ul>
        <button class="close-btn" @click="showModal = false">关闭</button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import {
  reactive,
  ref,
  computed,
  onMounted,
  nextTick,
  watch,
  onBeforeUnmount,
} from "vue";
import { sendMessageToHui } from "@/api/deepseekApi";

const STORAGE_KEY = "hui_chat_log";

// 本地存储键名
const STORAGE_STATS_KEY = "hui_chat_stats";
const showModal = ref(false);
// Stats 类型声明，确保所有字段都有默认值
interface Stats {
  firstTimestamp: number; // 首次使用时间戳
  totalChats: number; // 总对话次数
  activeDates: string[]; // 有发言的日期列表（yyyy‑mm‑dd）
  dailyChats: Record<string, number>; // 每日对话次数
  currentStreak: number; // 当前连续活跃天数
  longestStreak: number; // 历史最长连续活跃天数

  totalTime: number; // 累计使用时长（毫秒）
}

// 默认值，用于补齐本地存储中可能缺失的字段
const defaultStats: Stats = {
  firstTimestamp: Date.now(),
  totalChats: 0,
  activeDates: [],
  dailyChats: {},
  currentStreak: 0,
  longestStreak: 0,

  totalTime: 0,
};

// 从 localStorage 加载并合并默认值
function loadStats(): Stats {
  const saved = localStorage.getItem(STORAGE_STATS_KEY);
  if (saved) {
    try {
      const parsed = JSON.parse(saved);
      return { ...defaultStats, ...parsed };
    } catch {
      console.warn("加载统计数据失败，使用默认值");
    }
  }
  return { ...defaultStats };
}

// 保存到 localStorage
function saveStats() {
  localStorage.setItem(STORAGE_STATS_KEY, JSON.stringify(stats));
}

// 更新「活跃天数」及「连续活跃」逻辑
function updateActive(date: string) {
  if (!stats.activeDates.includes(date)) {
    stats.activeDates.push(date);
    updateStreak();
    saveStats(); // 持久化活跃天数变化
  }
}
function updateStreak() {
  const dates = [...stats.activeDates].sort();
  let curr = 0,
    max = stats.longestStreak,
    prevTs = 0;
  const todayStr = new Date().toISOString().slice(0, 10);
  dates.forEach((d) => {
    const ts = new Date(d).getTime();
    if (prevTs && ts - prevTs === 86400000) curr++;
    else curr = 1;
    max = Math.max(max, curr);
    prevTs = ts;
  });
  stats.currentStreak = dates[dates.length - 1] === todayStr ? curr : 0;
  stats.longestStreak = max;
  saveStats();
}

// 更新「每日对话次数」
function updateDaily(date: string) {
  stats.dailyChats[date] = (stats.dailyChats[date] || 0) + 1;
  saveStats(); // 持久化活跃天数变化
}

// 计算最活跃日
const mostActiveDayComputed = computed(() => {
  let day = "",
    max = 0;
  for (const [d, c] of Object.entries(stats.dailyChats)) {
    if (c > max) {
      max = c;
      day = d;
    }
  }
  return day || new Date().toISOString().slice(0, 10);
});

// 格式化总使用时长
function formatDuration(ms: number): string {
  const totalMin = Math.floor(ms / 60000);
  const h = Math.floor(totalMin / 60);
  const m = totalMin % 60;
  return h ? `${h} 小时 ${m} 分钟` : `${m} 分钟`;
}

// —— Vue 响应式状态 ——
const stats = reactive<Stats>(loadStats());
// 会话开始时间，用于计算本次时长
const sessionStart = Date.now();

interface ChatMsg {
  id: number;
  role: "user" | "bot";
  text: string;
  isError?: boolean;
  isEgg?: boolean;
}

const chatLog = ref<ChatMsg[]>(loadChatLog());
const input = ref("");
const loading = ref(false);
const msgList = ref<HTMLElement>();

const encourageEggs = [
  {
    file: "audio (0).mp3",
    text: "局之未终，焰之未熄。世事往往如此，输赢不过一息间。",
  },
  { file: "audio (1).mp3", text: "若心能如棋般冷静，是否就能不惧别离？" },
  {
    file: "audio (2).mp3",
    text: "一子错落，风云便改。可叹世人，总爱求个‘若当初’。",
  },
  {
    file: "audio (3).mp3",
    text: "命运是场漫长的棋，不在快与慢，而在落子的方向。",
  },
  {
    file: "audio (4).mp3",
    text: "别太在意成败了，局外人眼中的输赢，从来都不重要。",
  },
  { file: "audio (5).mp3", text: "我不是在等你，只是……棋还没落完。" },
  {
    file: "audio (6).mp3",
    text: "有时我会想——若当初那一步不落，也许局势就会不同吧。可惜啊，人生的棋盘从不容后悔，落子之后，只能微笑着看它燃成灰烬。",
  },
  {
    file: "audio (7).mp3",
    text: "他们总说我看得太远，其实我只是怕看得太近。太近的情感，会让人忘了自己站在何处。",
  },
  {
    file: "audio (8).mp3",
    text: "每一局的终点，都不只是输赢。有时，是人心的分岔口。",
  },
  {
    file: "audio (9).mp3",
    text: "你也觉得风冷吗？……别怕，靠近一点。焰不会伤人，若你心够静。",
  },
  {
    file: "audio (10).mp3",
    text: "你看着这片海时，会不会也觉得它像我？平静、无波，却藏着所有的光与深渊。",
  },
  {
    file: "audio (11).mp3",
    text: "我不常劝人，但若是你，我希望你能稍微放慢脚步。连焰，也该学会休息。",
  },
  {
    file: "audio (12).mp3",
    text: "我不是在等你，只是在听……听那一声久违的问候，能否再传回来。",
  },
  {
    file: "audio (13).mp3",
    text: "有些话不该说出口，但今晚的风很温柔，也许能替我说完。",
  },
  {
    file: "audio (14).mp3",
    text: "将自身置于棋局，才能算得更远。为此，我愿承受这离火焚身之苦。",
  },
  {
    file: "audio (15).mp3",
    text: "因果循环，终有报偿。昔日种下的‘因’，便由你我今日来见证这‘果’。",
  },
  {
    file: "audio (16).mp3",
    text: "即使燃尽己身，我也会为你照亮前方的长夜。这是我的承诺。",
  },
  {
    file: "audio (17).mp3",
    text: "若有一天我不在了，愿这缕离火，能代替我继续温暖你。",
  },
  {
    file: "audio (18).mp3",
    text: "当烈焰蒙蔽双眼，我反而能‘看’得更清。耳畔的潮音，指尖的颤动，还有……你的心意，都无比明晰。",
  },
  {
    file: "audio (19).mp3",
    text: "偶尔，也会想放下棋局与筹谋，只静静看着庭前的花开花落。这样的时光，有你相伴，似乎也不坏。",
  },
  {
    file: "audio (20).mp3",
    text: "若有一日，我真的燃尽己身，化作点点星火，但愿其中一簇，能永远温暖你的梦境。",
  },
];

function playVoice(name: string) {
  const audio = new Audio(`/voice/${name}`);
  audio.play().catch((e) => console.warn("音频播放失败：", e));
}

let lastEggTime = 0; // 记录最后一次触发彩蛋的时间戳
let coolDownPeriod = 1 * 60 * 1000; // 冷却1分钟（毫秒）
const triggeredVoices = new Set(
  JSON.parse(localStorage.getItem("triggeredVoices") || "[]")
);

async function sendMessage() {
  if (!input.value.trim()) return;
  if (stats.totalChats === 0 && !localStorage.getItem(STORAGE_STATS_KEY)) {
    stats.firstTimestamp = Date.now();
    saveStats();
  }
  const date = new Date().toISOString().slice(0, 10); // 每次都取最新“今天”
  stats.totalChats++;
  updateActive(date);
  updateDaily(date);
  saveStats();

  const userText = input.value;
  chatLog.value.push({
    id: Date.now(),
    role: "user",
    text: userText,
  });
  input.value = "";
  loading.value = true;

  try {
    //  throw new Error("测试错误");
    const history = chatLog.value.filter((msg) => !msg.isEgg && !msg.isError);
    const botReply = await sendMessageToHui(userText, history);
    chatLog.value.push({
      id: Date.now() + 1,
      role: "bot",
      text: botReply,
    });

    // —— 鼓励彩蛋：5% 概率触发 ——
    if (Date.now() - lastEggTime > coolDownPeriod && Math.random() < 0.05) {
      // 随机挑一条
      let voiceId = Math.floor(Math.random() * encourageEggs.length);
      const egg = encourageEggs[voiceId];

      // 记录触发过的语音编号
      triggeredVoices.add(voiceId || 0);

      // 保存到 localStorage
      localStorage.setItem(
        "triggeredVoices",
        JSON.stringify([...triggeredVoices])
      );

      // 播放对应语音（不带 .mp3 后缀）
      playVoice(egg.file);
      // 推入带标记的彩蛋消息
      chatLog.value.push({
        id: Date.now() + 2,
        role: "bot",
        text: `<p style="color: #4ea6ff; font-style: italic;font-weight: bold">${egg.text}</p>`,
        isEgg: true,
      });
      lastEggTime = Date.now();
    }
    // —— 彩蛋结束 ——
  } catch (e) {
    console.error(e);
    chatLog.value.push({
      id: Date.now() + 2,
      role: "bot",
      text: "API余额耗尽了，去b站提醒我充钱吧",
      isError: true,
    });
  } finally {
    loading.value = false;
    await scrollToBottom();
  }
}

function handleKeydown(e: KeyboardEvent) {
  if (e.key === "Enter") sendMessage();
}

function clearChat() {
  if (confirm("确定要清空全部对话吗？")) {
    chatLog.value = [
      {
        id: Date.now(),
        role: "bot",
        text: "焰有尽，而言无终。若你愿听，我便缓缓道来。",
      },
    ];
    localStorage.removeItem(STORAGE_KEY);
  }
}

function loadChatLog(): ChatMsg[] {
  const saved = localStorage.getItem(STORAGE_KEY);
  if (saved) {
    try {
      return JSON.parse(saved);
    } catch (e) {
      console.error("chatLog 解析失败：", e);
    }
  }
  return [
    {
      id: Date.now(),
      role: "bot",
      text: "焰有尽，而言无终。若你愿听，我便缓缓道来。",
    },
  ];
}

async function scrollToBottom() {
  await nextTick();
  if (msgList.value) {
    msgList.value.scrollTop = msgList.value.scrollHeight;
  }
}

watch(
  chatLog,
  async () => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(chatLog.value));
    await scrollToBottom();
  },
  { deep: true }
);

function handleBeforeUnload() {
  stats.totalTime += Date.now() - sessionStart;
  saveStats();
}

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
  scrollToBottom();
  window.addEventListener("beforeunload", handleBeforeUnload);
  Imgtimer = window.setInterval(() => {
    currentIndex.value =
      (currentIndex.value + 1) % Math.max(1, randomFive.value.length);
  }, 5200);
});

onBeforeUnmount(() => {
  window.removeEventListener("beforeunload", handleBeforeUnload);
  if (Imgtimer) clearInterval(Imgtimer);
});
</script>

<style scoped lang="scss">
@import url("https://fonts.googleapis.com/css2?family=Noto+Sans+SC:wght@300;400;700&display=swap");

/* ====== 尤诺风格 · 聊天页 ====== */
/* 颜色主轴
   主底：深夜藍 #06122a
   月光：#5fc3ff / #9fe8ff（冷蓝）
   银灰高光：#dff8ff / #eaf7ff
   极浅金色点缀：#ffd78a（仅作极细点缀）
*/
.chat-page {
  padding-top: 64px;
  min-height: 100vh;
  font-family: "Noto Sans SC", "PingFang SC", "Helvetica Neue", Arial,
    sans-serif;
  color: #eaf7ff;
  display: flex;
  flex-direction: column;
  background: linear-gradient(180deg, #051025 0%, #020611 100%); /* 夜色底 */
  position: relative;
  overflow-x: hidden;

  /* 轻微月光气流层 */
  &::before {
    content: "";
    position: fixed;
    inset: 0;
    background: radial-gradient(
        600px 160px at 10% 8%,
        rgba(95, 195, 255, 0.04),
        transparent 12%
      ),
      radial-gradient(
        420px 120px at 78% 18%,
        rgba(143, 217, 255, 0.03),
        transparent 12%
      );
    pointer-events: none;
    z-index: 0;
    mix-blend-mode: screen;
    filter: blur(6px);
  }

  /* 顶部/尾部小颗粒，微弱星尘 */
  &::after {
    content: "";
    position: fixed;
    inset: 0;
    background-image: radial-gradient(
        circle at 8% 14%,
        rgba(255, 255, 255, 0.02) 0.5px,
        transparent 1px
      ),
      radial-gradient(
        circle at 88% 26%,
        rgba(255, 223, 138, 0.01) 0.6px,
        transparent 1px
      ),
      radial-gradient(
        circle at 60% 78%,
        rgba(200, 240, 255, 0.015) 0.5px,
        transparent 1px
      );
    background-size: 160px 160px, 220px 220px, 200px 200px;
    pointer-events: none;
    z-index: 0;
    mix-blend-mode: screen;
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
        rgba(2, 6, 14, 0.12),
        rgba(2, 6, 14, 0.28)
      );
      pointer-events: none;
      z-index: 1;
      mix-blend-mode: multiply;
    }

    .carousel-image {
      position: absolute;
      width: 100%;
      height: 100%;
      object-fit: cover;
      opacity: 0;
      filter: blur(0.8px) saturate(0.9) contrast(0.96);
      transition: opacity 560ms ease,
        transform 760ms cubic-bezier(0.2, 0.9, 0.2, 1);

      &.active {
        opacity: 1;
        transform: scale(1);
      }
    }
  }

  .carousel2 {
    display: none;
  }

  .chat-container {
    flex: 1;
    width: 920px;
    max-width: calc(100% - 32px);
    margin: 0 auto;
    padding: 24px;
    display: flex;
    flex-direction: column;
    gap: 16px;
    z-index: 9;
    position: relative;
  }

  /* 统计面板：玻璃感 + 月光高光 */
  .stats-panel {
    display: flex;
    align-items: center;
    background: linear-gradient(
      180deg,
      rgba(8, 14, 22, 0.58),
      rgba(6, 10, 18, 0.54)
    );
    padding: 10px 16px;
    border-radius: 12px;
    font-size: 14px;
    color: #dff8ff;
    justify-content: space-around;
    box-shadow: 0 12px 34px rgba(3, 8, 18, 0.6);
    border: 1px solid rgba(143, 217, 255, 0.03);
    backdrop-filter: blur(6px) saturate(0.98);

    .stat-item {
      .label {
        font-size: 12px;
        color: rgba(222, 245, 255, 0.7);
        margin-bottom: 4px;
      }

      span {
        color: #9fe8ff;
        font-weight: 700;
        font-size: 15px;
        text-shadow: 0 0 6px rgba(95, 195, 255, 0.06);
      }
    }

    .detail-btn {
      background: transparent;
      border: 1px solid rgba(143, 217, 255, 0.06);
      border-radius: 8px;
      color: #dff8ff;
      padding: 6px 12px;
      cursor: pointer;
      font-size: 13px;
      transition: background 0.16s ease, box-shadow 0.16s ease, transform 0.12s;

      &:hover {
        background: rgba(95, 195, 255, 0.04);
        box-shadow: 0 10px 26px rgba(78, 160, 255, 0.06);
        transform: translateY(-2px);
      }

      &:focus-visible {
        outline: none;
        box-shadow: 0 0 0 6px rgba(95, 195, 255, 0.04);
      }
    }
  }

  .messages {
    flex: 1;
    overflow-y: auto;
    padding: 12px 0 140px;
    overscroll-behavior: contain;
    scroll-behavior: smooth;
    max-height: 72vh;
    z-index: 6;
  }

  /* 消息气泡：尤诺风格（冷蓝月光 / 银灰内光） */
  .message {
    display: flex;
    align-items: flex-start;
    margin-bottom: 14px;
    color: #eaf7ff;
    position: relative;

    &.user {
      flex-direction: row-reverse;
    }

    &.error .bubble {
      background: linear-gradient(
        180deg,
        rgba(80, 28, 34, 0.16),
        rgba(52, 18, 20, 0.12)
      );
      border: 1px solid rgba(200, 60, 70, 0.12);
      box-shadow: 0 8px 26px rgba(80, 30, 30, 0.06);
    }

    &.egg .bubble {
      border: 1px solid rgba(143, 217, 255, 0.06);
      box-shadow: 0 8px 26px rgba(78, 160, 255, 0.06);
      transition: all 0.28s cubic-bezier(0.2, 0.9, 0.2, 1);
    }

    .avatar {
      width: 48px;
      height: 48px;
      border-radius: 50%;
      margin: 0 10px;
      background-size: cover;
      background-position: center;
      flex-shrink: 0;
      z-index: 10;
      box-shadow: 0 8px 28px rgba(2, 6, 12, 0.6);
      border: 1px solid rgba(255, 255, 255, 0.03);
      background-clip: padding-box;

      &.bot {
        /* 机器人头像：冷蓝内光 + 轻微反射 */
        background-image: url("@/assets/avatar/changli.png");
        box-shadow: 0 10px 34px rgba(78, 160, 255, 0.06),
          inset 0 1px 0 rgba(220, 245, 255, 0.02);
        border: 2px solid rgba(143, 217, 255, 0.06);
        transform: scaleX(-1);
      }

      &.user {
        background: linear-gradient(180deg, #08101a, #071018);
        box-shadow: 0 8px 22px rgba(2, 6, 12, 0.6);
        border: 1px solid rgba(143, 217, 255, 0.02);
      }
    }

    .bubble {
      max-width: 78%;
      background: linear-gradient(
        180deg,
        rgba(6, 12, 22, 0.94),
        rgba(8, 14, 26, 0.96)
      );
      border: 1px solid rgba(143, 217, 255, 0.04);
      padding: 14px 16px;
      border-radius: 16px;
      line-height: 1.7;
      word-break: break-word;
      box-shadow: 0 10px 30px rgba(2, 6, 14, 0.6);
      transition: box-shadow 0.18s, transform 0.12s, background 0.12s;
      color: #eaf7ff;
      position: relative;
      overflow: visible;
      backdrop-filter: blur(2px) saturate(1);

      /* 月光内衬（轻薄高光条） */
      &::after {
        content: "";
        position: absolute;
        left: 8px;
        right: 8px;
        top: 6px;
        height: 6px;
        border-radius: 6px;
        background: linear-gradient(
          90deg,
          rgba(95, 195, 255, 0.06),
          rgba(223, 248, 255, 0.03)
        );
        pointer-events: none;
        mix-blend-mode: screen;
        opacity: 0.95;
        filter: blur(1px);
      }

      &:hover {
        box-shadow: 0 18px 44px rgba(2, 6, 14, 0.7),
          0 0 30px rgba(95, 195, 255, 0.04);
        transform: translateY(-2px);
      }

      &.loading {
        color: rgba(234, 247, 255, 0.92);
        opacity: 0.98;
      }

      /* bot 消息视觉尾巴：左侧微弧，并带月光边 */
      .message.bot & {
        border-radius: 16px 16px 16px 6px;
        background: linear-gradient(
          160deg,
          rgba(9, 20, 34, 0.98),
          rgba(6, 12, 22, 0.96)
        );
        box-shadow: 0 12px 36px rgba(6, 10, 18, 0.6),
          inset 0 1px 0 rgba(223, 248, 255, 0.01);
      }

      /* user 消息右侧尾巴 */
      .message.user & {
        border-radius: 16px 16px 6px 16px;
        background: linear-gradient(
          200deg,
          rgba(8, 16, 28, 0.98),
          rgba(7, 14, 26, 0.96)
        );
      }

      .dots {
        display: inline-flex;
        align-items: center;
        margin-left: 6px;

        .dot {
          opacity: 0;
          font-size: 16px;
          animation: blink 1s infinite;
          color: #9fe8ff;

          &:nth-child(1) {
            animation-delay: 0s;
          }
          &:nth-child(2) {
            animation-delay: 0.18s;
          }
          &:nth-child(3) {
            animation-delay: 0.36s;
          }
        }

        @keyframes blink {
          0%,
          100% {
            opacity: 0;
            transform: translateY(0);
          }
          50% {
            opacity: 1;
            transform: translateY(-4px);
            color: #9fe8ff;
          }
        }
      }
    }
  }

  /* 输入区：月匣感（玻璃 + 月光按钮） */
  .input-area {
    position: sticky;
    bottom: 12px;
    display: flex;
    align-items: center;
    background: linear-gradient(
      180deg,
      rgba(6, 10, 16, 0.92),
      rgba(8, 12, 18, 0.94)
    );
    backdrop-filter: blur(8px) saturate(1.02);
    padding: 12px;
    gap: 12px;
    z-index: 30;
    border-radius: 14px;
    box-shadow: 0 18px 44px rgba(2, 6, 14, 0.72);
    border: 1px solid rgba(143, 217, 255, 0.04);

    textarea {
      flex: 1;
      padding: 12px 16px;
      background: linear-gradient(
        180deg,
        rgba(8, 12, 18, 0.92),
        rgba(6, 10, 16, 0.9)
      );
      border: 1px solid rgba(143, 217, 255, 0.04);
      color: #eaf7ff;
      font-size: 15px;
      line-height: 1.45;
      outline: none;
      resize: none;
      overflow: hidden;
      min-height: 48px;
      max-height: 160px;
      border-radius: 10px;
      box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.02);
      transition: box-shadow 0.12s, border-color 0.12s, transform 0.12s;

      &::placeholder {
        color: rgba(234, 247, 255, 0.22);
      }

      &:focus {
        border-color: rgba(95, 195, 255, 0.26);
        box-shadow: 0 0 0 6px rgba(95, 195, 255, 0.04);
        transform: translateY(-1px);
      }
    }

    .btn-group {
      display: flex;
      gap: 8px;

      button {
        display: inline-flex;
        align-items: center;
        justify-content: center;
        width: 40px;
        height: 40px;
        padding: 0;
        border: none;
        border-radius: 10px;
        background: linear-gradient(
          180deg,
          rgba(12, 16, 22, 0.6),
          rgba(8, 12, 18, 0.6)
        );
        color: #dff8ff;
        cursor: pointer;
        transition: transform 0.12s, box-shadow 0.12s, background 0.12s;
        box-shadow: 0 6px 18px rgba(2, 6, 12, 0.6);
        border: 1px solid rgba(143, 217, 255, 0.03);

        &:hover {
          transform: translateY(-2px);
          background: rgba(95, 195, 255, 0.04);
          box-shadow: 0 12px 28px rgba(78, 160, 255, 0.06);
        }

        &:active {
          transform: translateY(0);
        }

        &:disabled {
          opacity: 0.5;
          cursor: not-allowed;
        }
      }

      .clear-btn {
        font-size: 16px;
        line-height: 1;
      }
    }

    .send-btn {
      padding: 0 22px;
      height: 40px;
      border: none;
      border-radius: 20px;
      background: linear-gradient(135deg, #6fc9ff 0%, #2f92ff 100%);
      color: #02101a;
      font-weight: 700;
      font-size: 15px;
      cursor: pointer;
      box-shadow: 0 12px 36px rgba(47, 146, 255, 0.12);
      transition: transform 0.12s, box-shadow 0.18s, filter 0.12s;

      &:hover:not(:disabled) {
        transform: translateY(-3px);
        box-shadow: 0 18px 46px rgba(47, 146, 255, 0.16);
        filter: saturate(1.06);
      }

      &:disabled {
        opacity: 0.5;
        cursor: not-allowed;
        box-shadow: none;
      }

      &:focus-visible {
        outline: none;
        box-shadow: 0 0 0 8px rgba(95, 195, 255, 0.06);
      }
    }

    .Alldetail-btn {
      display: none;
      margin-left: 4px;
      background: transparent;
      border: 1px solid rgba(143, 217, 255, 0.06);
      border-radius: 6px;
      padding: 6px 10px;
      color: #dff8ff;
      font-size: 13px;
      cursor: pointer;

      &:hover {
        background: rgba(95, 195, 255, 0.03);
        box-shadow: 0 6px 14px rgba(78, 160, 255, 0.04);
      }
    }
  }

  /* 模态框（玻璃月匣） */
  .modal-overlay {
    position: fixed;
    inset: 0;
    background: linear-gradient(
      180deg,
      rgba(3, 6, 12, 0.78),
      rgba(6, 10, 18, 0.9)
    );
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 2000;
    padding: 16px;

    .modal-content {
      width: 380px;
      max-width: 100%;
      background: linear-gradient(
        180deg,
        rgba(6, 10, 16, 0.98),
        rgba(8, 12, 18, 0.98)
      );
      backdrop-filter: blur(8px) saturate(1.02);
      border-radius: 14px;
      padding: 20px;
      color: #eaf7ff;
      box-shadow: 0 20px 60px rgba(2, 6, 14, 0.9);
      border: 1px solid rgba(143, 217, 255, 0.04);
      animation: fadeInUp 220ms ease;
      position: relative;

      &::before {
        content: "☾";
        position: absolute;
        left: 14px;
        top: 10px;
        font-size: 14px;
        color: rgba(143, 217, 255, 0.08);
        pointer-events: none;
      }

      h3 {
        margin: 0 0 12px 0;
        font-size: 18px;
        font-weight: 700;
        text-align: center;
        color: #9fe8ff;
        padding-bottom: 8px;
        border-bottom: 1px dashed rgba(143, 217, 255, 0.04);
      }

      .detail-list {
        list-style: none;
        padding: 0;
        margin: 12px 0 18px;
        line-height: 1.6;
        font-size: 14px;
        color: #dff8ff;

        li {
          margin-bottom: 8px;
          padding-left: 6px;

          &:nth-child(odd) {
            color: rgba(223, 248, 255, 0.9);
          }
          &:last-child {
            margin-bottom: 0;
          }
        }
      }

      .close-btn {
        display: block;
        margin: 0 auto;
        padding: 8px 20px;
        background: linear-gradient(135deg, #6fc9ff, #2f92ff);
        color: #02101a;
        border: none;
        border-radius: 10px;
        cursor: pointer;
        font-weight: 700;
        box-shadow: 0 12px 36px rgba(47, 146, 255, 0.12);
        transition: transform 0.12s ease, box-shadow 0.14s ease;

        &:hover {
          transform: translateY(-3px);
          box-shadow: 0 18px 46px rgba(47, 146, 255, 0.16);
        }
        &:active {
          transform: translateY(-1px) scale(0.996);
        }
        &:focus-visible {
          outline: none;
          box-shadow: 0 0 0 8px rgba(95, 195, 255, 0.06);
        }
      }
    }

    @keyframes fadeInUp {
      from {
        opacity: 0;
        transform: translateY(8px) scale(0.995);
      }
      to {
        opacity: 1;
        transform: translateY(0) scale(1);
      }
    }

    @media (max-width: 480px) {
      .modal-content {
        width: 100%;
        padding: 16px;
        border-radius: 12px;
        h3 {
          font-size: 16px;
        }
        .close-btn {
          width: 100%;
          padding: 10px 14px;
          border-radius: 8px;
        }
      }
    }
  }

  /* 响应式调整（保持原来逻辑，仅调整间距与头像大小） */
  @media (max-width: 768px) {
    .chat-container {
      width: 100%;
      padding: 12px;
      padding-top: 20px;

      .carousel1 {
        display: none;
      }
      .carousel2 {
        display: block;
      }
      .stats-panel {
        display: none;
      }
    }

    .bubble {
      padding: 10px 12px;
      font-size: 14px;
      max-width: 86%;
    }

    .avatar {
      width: 36px;
      height: 36px;
    }

    .input-area {
      flex-direction: column;
      align-items: stretch;

      textarea {
        width: 100%;
      }
      button {
        width: 100%;
      }
      .Alldetail-btn {
        display: block;
      }
    }
  }
}

/* ====== 轻量动效关键帧 ====== */
@keyframes tide-flow {
  0% {
    transform: translateY(0) translateX(0) rotate(-2deg);
    opacity: 0.9;
  }
  50% {
    transform: translateY(-6px) translateX(-4px) rotate(2deg);
    opacity: 1;
  }
  100% {
    transform: translateY(0) translateX(0) rotate(-2deg);
    opacity: 0.9;
  }
}

@keyframes ripple {
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
