<script setup lang="ts">
import { onMounted, onUnmounted, reactive, ref, watch } from "vue";

type Wish = {
  id: string;
  name?: string;
  text: string;
  createdAt: number;
};

const STORAGE_KEY = "hangawi-wishes";

const state = reactive({
  name: "",
  text: "",
  wishes: [] as Wish[],
  charLimit: 100,
  submitting: false,
  showToast: false,
  toastMsg: "",
});

const remaining = ref(100);
watch(
  () => state.text,
  (v) => (remaining.value = state.charLimit - v.length)
);

const canvasEl = ref<HTMLCanvasElement | null>(null);
let rafId = 0;

// 간단한 컨페티(종이조각) 애니메이션
type Confetti = {
  x: number;
  y: number;
  r: number;
  vx: number;
  vy: number;
  a: number;
  va: number;
};
let confettis: Confetti[] = [];

function spawnConfetti(n = 80) {
  const w = canvasEl.value!.width;
  const h = canvasEl.value!.height;
  confettis = Array.from({ length: n }).map(() => ({
    x: Math.random() * w,
    y: -10 - Math.random() * 100,
    r: 2 + Math.random() * 4,
    vx: -0.5 + Math.random(),
    vy: 1 + Math.random() * 2,
    a: Math.random() * Math.PI * 2,
    va: -0.05 + Math.random() * 0.1,
  }));
  animate();
}

function animate() {
  const ctx = canvasEl.value?.getContext("2d");
  if (!ctx || !canvasEl.value) return;
  const w = (canvasEl.value.width = canvasEl.value.clientWidth);
  const h = (canvasEl.value.height = canvasEl.value.clientHeight);

  ctx.clearRect(0, 0, w, h);
  confettis.forEach((c) => {
    c.x += c.vx;
    c.y += c.vy;
    c.a += c.va;

    ctx.save();
    ctx.translate(c.x, c.y);
    ctx.rotate(c.a);
    ctx.fillStyle = `hsl(${(c.x + c.y) % 360}, 80%, 60%)`;
    ctx.fillRect(-c.r, -c.r, c.r * 2, c.r * 2);
    ctx.restore();
  });
  confettis = confettis.filter((c) => c.y < h + 20);
  if (confettis.length > 0) {
    rafId = requestAnimationFrame(animate);
  }
}

function loadWishes() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (raw) state.wishes = JSON.parse(raw);
  } catch {}
}

function saveWishes() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(state.wishes));
}

function toast(msg: string) {
  state.toastMsg = msg;
  state.showToast = true;
  setTimeout(() => (state.showToast = false), 1800);
}

function submitWish() {
  if (!state.text.trim()) {
    toast("소원을 적어주세요!");
    return;
  }
  if (state.text.length > state.charLimit) {
    toast(`소원은 최대 ${state.charLimit}자까지 가능해요`);
    return;
  }
  state.submitting = true;
  const wish: Wish = {
    id: crypto.randomUUID(),
    name: state.name.trim() || undefined,
    text: state.text.trim(),
    createdAt: Date.now(),
  };
  state.wishes.unshift(wish);
  saveWishes();
  state.text = "";
  spawnConfetti();
  toast("소원을 하늘에 띄웠어요 🌕");
  state.submitting = false;
}

function removeWish(id: string) {
  state.wishes = state.wishes.filter((w) => w.id !== id);
  saveWishes();
  toast("소원을 조용히 내려두었어요");
}

onMounted(() => {
  loadWishes();
  // 리사이즈에 맞춰 캔버스 리프레시
  const onResize = () => {
    if (canvasEl.value) {
      canvasEl.value.width = canvasEl.value.clientWidth;
      canvasEl.value.height = canvasEl.value.clientHeight;
    }
  };
  window.addEventListener("resize", onResize);
  onResize();
});

onUnmounted(() => {
  cancelAnimationFrame(rafId);
});
</script>

<template>
  <div class="page">
    <!-- 배경 장식 -->
    <div class="sky">
      <div class="stars" aria-hidden="true"></div>
      <div class="moon" aria-hidden="true">
        <div class="crater c1"></div>
        <div class="crater c2"></div>
        <div class="crater c3"></div>
      </div>
      <div class="cloud c-left" aria-hidden="true"></div>
      <div class="cloud c-right" aria-hidden="true"></div>
      <canvas ref="canvasEl" class="confetti" aria-hidden="true"></canvas>
    </div>

    <!-- 헤더 -->
    <header class="header">
      <h1>🌕 한가위 복 많이 받으세요!</h1>
      <p class="subtitle">
        풍성한 보름달처럼 마음도 가득 차오르길 바랍니다. 아래에
        <strong>소원</strong>을 적고 하늘에 띄워보세요.
      </p>
    </header>

    <!-- 입력 폼 -->
    <section class="panel">
      <form
        @submit.prevent="submitWish"
        class="wish-form"
        aria-label="소원 입력 폼"
      >
        <div class="field inline">
          <label for="name">이름(선택)</label>
          <input
            id="name"
            v-model="state.name"
            type="text"
            inputmode="text"
            placeholder="예) 홍길동"
            autocomplete="name"
          />
        </div>

        <div class="field">
          <label for="wish">소원</label>
          <textarea
            id="wish"
            v-model="state.text"
            :maxlength="state.charLimit"
            rows="3"
            placeholder="가족 건강, 시험 합격, 프로젝트 성공… 마음속 소원을 적어보세요"
          ></textarea>
          <div class="hint">
            <span>남은 글자 수: {{ remaining }}</span>
          </div>
        </div>

        <div class="actions">
          <button
            class="btn primary"
            type="submit"
            :disabled="state.submitting"
          >
            🌠 소원 빌기
          </button>
        </div>
      </form>
    </section>

    <!-- 소원 등(리스트) -->
    <section class="panel">
      <h2>띄운 소원 등</h2>
      <p v-if="state.wishes.length === 0" class="empty">
        아직 띄운 소원이 없어요. 첫 소원을 적어보세요!
      </p>
      <ul v-else class="wish-list">
        <li v-for="w in state.wishes" :key="w.id" class="wish-item">
          <div class="lantern" aria-hidden="true"></div>
          <div class="content">
            <div class="meta">
              <span class="name">{{ w.name ?? "익명" }}</span>
              <time :datetime="new Date(w.createdAt).toISOString()">
                {{ new Date(w.createdAt).toLocaleString() }}
              </time>
            </div>
            <p class="text">{{ w.text }}</p>
          </div>
          <button
            class="icon-btn"
            @click="removeWish(w.id)"
            aria-label="소원 삭제"
          >
            🗑️
          </button>
        </li>
      </ul>
    </section>

    <!-- 토스트 -->
    <div
      class="toast"
      v-show="state.showToast"
      role="status"
      aria-live="polite"
    >
      {{ state.toastMsg }}
    </div>

    <!-- 푸터 -->
    <footer class="footer">
      <small>행복한 한가위 보내세요 · {{ new Date().getFullYear() }}</small>
    </footer>
  </div>
</template>

<style scoped>
/* 레이아웃 */
.page {
  min-height: 100dvh;
  color: #f7f7fb;
  background: linear-gradient(180deg, #0a0f2c 0%, #0c173e 50%, #12204f 100%);
  display: grid;
  grid-template-rows: auto auto auto 1fr auto;
  gap: 16px;
  position: relative;
  overflow-x: hidden;
  padding-bottom: 32px;
}
.header {
  text-align: center;
  padding-top: 24px;
}
h1 {
  margin: 0 0 8px;
  font-size: clamp(24px, 3.4vw, 40px);
  letter-spacing: 0.5px;
}
.subtitle {
  margin: 0 auto;
  opacity: 0.9;
  max-width: 720px;
}
.panel {
  width: min(960px, 92%);
  margin: 0 auto;
  background: rgba(255, 255, 255, 0.06);
  border: 1px solid rgba(255, 255, 255, 0.12);
  border-radius: 16px;
  backdrop-filter: blur(6px);
  padding: 16px 16px 8px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.25);
}

/* 하늘/달/구름 */
.sky {
  position: absolute;
  inset: 0;
  overflow: hidden;
  pointer-events: none;
}
.confetti {
  position: absolute;
  inset: 0;
}
.stars {
  position: absolute;
  inset: 0;
  background-image: radial-gradient(2px 2px at 20% 30%, #fff8 50%, #fff0 51%),
    radial-gradient(1.5px 1.5px at 70% 20%, #fff8 50%, #fff0 51%),
    radial-gradient(1.5px 1.5px at 40% 80%, #fff8 50%, #fff0 51%),
    radial-gradient(1.5px 1.5px at 85% 60%, #fff8 50%, #fff0 51%),
    radial-gradient(1.5px 1.5px at 10% 70%, #fff8 50%, #fff0 51%);
  opacity: 0.65;
}
.moon {
  --size: min(22vw, 220px);
  position: absolute;
  top: 40px;
  right: 8%;
  width: var(--size);
  height: var(--size);
  border-radius: 50%;
  background: radial-gradient(circle at 30% 30%, #fff8, #fff1 40%),
    radial-gradient(circle at 50% 50%, #ffe9b8, #ffcf70 60%, #f6b94d 100%);
  box-shadow: 0 0 40px 12px #ffd27855, 0 0 120px 24px #ffc34d33;
}
.moon .crater {
  position: absolute;
  border-radius: 50%;
  background: radial-gradient(
    circle at 30% 30%,
    #0002,
    #00000015 60%,
    #0000 70%
  );
}
.moon .c1 {
  width: 18%;
  height: 18%;
  left: 20%;
  top: 24%;
}
.moon .c2 {
  width: 12%;
  height: 12%;
  right: 22%;
  top: 28%;
}
.moon .c3 {
  width: 15%;
  height: 15%;
  left: 46%;
  bottom: 18%;
}

.cloud {
  position: absolute;
  width: 220px;
  height: 70px;
  background: #fff2;
  border-radius: 35px;
  filter: blur(1px);
  animation: drift 40s linear infinite;
}
.c-left {
  left: -240px;
  top: 120px;
  animation-delay: -5s;
}
.c-right {
  right: -240px;
  top: 200px;
  animation-delay: -18s;
}
@keyframes drift {
  from {
    transform: translateX(0);
  }
  to {
    transform: translateX(120vw);
  }
}

/* 폼 */
.wish-form {
  display: grid;
  gap: 12px;
}
.field label {
  display: block;
  font-weight: 600;
  margin-bottom: 6px;
}
.field input,
.field textarea {
  width: 100%;
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.25);
  background: rgba(255, 255, 255, 0.08);
  color: #fff;
  padding: 10px 12px;
  outline: none;
}
.field input::placeholder,
.field textarea::placeholder {
  color: #e8e8f0aa;
}
.field textarea {
  resize: vertical;
}
.field .hint {
  margin-top: 4px;
  font-size: 12px;
  opacity: 0.8;
}
.field.inline {
  display: grid;
  grid-template-columns: 110px 1fr;
  gap: 10px;
  align-items: center;
}

.actions {
  display: flex;
  gap: 8px;
  justify-content: flex-end;
}
.btn {
  border: none;
  padding: 10px 14px;
  border-radius: 12px;
  cursor: pointer;
  font-weight: 700;
}
.btn.primary {
  background: linear-gradient(90deg, #ffb64d, #ff8b3d);
  color: #221611;
  box-shadow: 0 8px 18px #ff9a3d33;
}
.btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* 소원 리스트 + 연등 */
h2 {
  margin: 4px 0 8px;
}
.empty {
  opacity: 0.9;
  margin: 8px 0 12px;
}
.wish-list {
  list-style: none;
  margin: 8px 0 12px;
  padding: 0;
  display: grid;
  gap: 12px;
}
.wish-item {
  position: relative;
  display: grid;
  grid-template-columns: 56px 1fr auto;
  gap: 12px;
  align-items: center;
  padding: 12px;
  border-radius: 14px;
  background: rgba(0, 0, 0, 0.25);
  border: 1px solid rgba(255, 255, 255, 0.12);
  overflow: hidden;
}
.lantern {
  width: 44px;
  height: 56px;
  border-radius: 12px 12px 16px 16px;
  background: radial-gradient(circle at 50% 10%, #ffffffaa, #fff0 60%),
    linear-gradient(180deg, #ffcf7a, #ff9a3d 55%, #ff7b3d 100%);
  box-shadow: 0 6px 14px #ff8b3d33, 0 -2px 12px inset #ffffff40;
  animation: floatY 6s ease-in-out infinite;
}
@keyframes floatY {
  0%,
  100% {
    transform: translateY(0px);
  }
  50% {
    transform: translateY(-6px);
  }
}
.content .meta {
  font-size: 12px;
  opacity: 0.9;
  display: flex;
  gap: 8px;
  align-items: baseline;
}
.content .name {
  font-weight: 700;
}
.content .text {
  margin: 4px 0 0;
  line-height: 1.5;
}
.icon-btn {
  background: transparent;
  border: none;
  font-size: 18px;
  cursor: pointer;
  color: #fff;
  opacity: 0.9;
}
.icon-btn:hover {
  opacity: 1;
}

/* 토스트 */
.toast {
  position: fixed;
  left: 50%;
  bottom: 24px;
  transform: translateX(-50%);
  background: rgba(0, 0, 0, 0.65);
  border: 1px solid rgba(255, 255, 255, 0.18);
  color: #fff;
  border-radius: 12px;
  padding: 10px 14px;
  z-index: 50;
}

/* 푸터 */
.footer {
  text-align: center;
  opacity: 0.9;
  margin-top: 8px;
}
</style>
