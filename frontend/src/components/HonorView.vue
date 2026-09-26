<script setup>
// 荣誉墙：两种呈现 —— ①「流动展厅」奖状多行反向跑马灯（默认）②「荣誉殿堂」证书网格。
// 公开免登录，内容由管理员在后台维护。
//
// —— 跑马灯是无缝循环，靠一条数学恒等式：**轨道恰好是两份等宽的副本**，动画从 0 位移到 -50%。
// 两个必须守住的点（写错就是"看着在动但其实是坏的"）：
//   ① 卡片间距用**每张卡的 margin-right**，不能用容器的 flex gap。
//      gap 只在相邻项之间插入、不在两份副本的接缝处出现，轨道宽 = 2kh·W + G 而不是 2kh·W，
//      -50% 就差了半个间距 → 每滚一圈画面横跳一下。用 margin-right，半份宽精确等于 n·(w+gap)。
//   ② 半份必须**铺满视口**，否则滚到接缝处会露出一段空白。重复次数 k 由实测视口宽算出来。
//
// 卡片宽/间距/图盒高都由 JS 常量给出、以 CSS 变量喂给样式表（单一事实来源），
// 避免"CSS 里写 228px、JS 里算 232px"这种漂移 —— 那种漂移同样会破坏 -50% 的等式。
//
// 装饰元素（顶边色条 .haccent / 奖牌点）一律 position:absolute，不进 flex 流、不改变卡片外宽，
// 所以它们碰不到上面那条宽度等式。切换视图靠 v-if，跑马灯卸载时 .marquee/.hcard 整体不在场，
// 网格用的是另一套类名（.hall-grid/.gcard），两套互不污染。
import { ref, computed, onMounted, onBeforeUnmount, watch } from 'vue';
import { useRouter } from 'vue-router';
import { api } from '../api.js';
import auth from '../auth.js';
import { openImage } from '../utils/imageViewer.js';

const router = useRouter();

const list = ref([]);
const loading = ref(true);
const err = ref('');

// —— 几何常量（整数 px）——
const GAP = 16;             // 卡片间距，做进卡片的 margin-right
const SPEED = 42;           // 基准像素速度 px/s（各行按 SPEED_MULT 微差，避免整墙同频）
const SPEED_MULT = [1, 0.82, 1.18];
const PER_ROW = 6;          // 每行条数的分档基数

const rowsRef = ref(null);
const vw = ref(1200);       // 行视口实测宽（ResizeObserver）
let ro = null;

// 桌面/窄屏两档卡片宽（用实测宽判定，不在 CSS 里再写一份 @media）
const narrow = computed(() => vw.value <= 768);
const cardW = computed(() => (narrow.value ? 150 : 228));
const boxH = computed(() => (narrow.value ? 96 : 132));
const W = computed(() => cardW.value + GAP); // 每张卡占的外宽

// 无障碍：系统开了"减弱动态效果"就彻底不滚，退化成可手动横滚的静态行（内容一张不少）
const reduced = ref(false);
let mq = null;
const onMq = (e) => { reduced.value = e.matches; };

const rowCount = computed(() => {
  const n = list.value.length;
  if (!n) return 0;
  return Math.min(narrow.value ? 2 : 3, Math.max(1, Math.ceil(n / PER_ROW)));
});

// 分行用**轮转**（i % R）而不是切块：切块会让第一行独占全部最高优先项、最后一行吃剩饭；
// 轮转把排序权重均摊到各行，观感等重，条数差也天然 ≤1
const buckets = computed(() => {
  const R = rowCount.value;
  const out = Array.from({ length: R }, () => []);
  list.value.forEach((h, i) => out[i % R].push(h));
  return out;
});

const lanes = computed(() => buckets.value.map((bucket, i) => {
  if (!bucket.length) return null;
  // 半份重复到"至少铺满视口 + 一张卡"的余量，保证滚到任意相位都不会露空
  const k = reduced.value ? 1 : Math.max(1, Math.ceil((vw.value + W.value) / (bucket.length * W.value)));
  const half = [];
  for (let r = 0; r < k; r++) half.push(...bucket);
  const halfW = half.length * W.value;
  const speed = SPEED * SPEED_MULT[i % SPEED_MULT.length];
  return {
    i, half, halfW, n: bucket.length,
    // 恒定**像素**速度：若所有行用同一个时长，条目多的行会明显跑得更快、像在互相追
    dur: +(halfW / speed).toFixed(2),
    rev: i % 2 === 1,
  };
}).filter(Boolean));

const copies = computed(() => (reduced.value ? 1 : 2)); // 降级时只渲染一份
const trackVars = computed(() => ({ '--hw': `${cardW.value}px`, '--hg': `${GAP}px`, '--bh': `${boxH.value}px` }));

// —— 荣誉等级（视觉分组，不替代原始 award_level 文案）——
// 按「授予范围」分三档：国家/国际 = 金，省部/区域 = 银，市校院/其他 = 铜。
// award_level 可能只写了"一等奖"，故连标题一起匹配；原始文案在卡片上照常全显，分档只是配色。
const TIER_RE = [
  ['gold', /(国际|国家|全国|世界|全球|洲际)/],
  ['silver', /(省|部|大区|赛区|华东|华北|华南|华中|西南|西北|东北|中南)/],
  ['bronze', /(市|州|地区|校|院|区|县)/],
];
const tierOf = (h) => {
  const s = `${h.award_level || ''} ${h.title || ''}`;
  return TIER_RE.find(([, re]) => re.test(s))?.[0] || 'bronze';
};

// —— 视图模式：流动展厅（默认）/ 荣誉殿堂网格。偏好记住，但默认永远是 flow（探针口径）——
const mode = ref(localStorage.getItem('honor_mode') === 'hall' ? 'hall' : 'flow');
watch(mode, (v) => localStorage.setItem('honor_mode', v));

const stat = computed(() => {
  const n = list.value.length;
  const top = list.value.filter((h) => /国家|国际/.test(h.award_level || '')).length;
  const years = new Set(
    list.value.map((h) => (h.award_date || '').slice(0, 4)).filter((y) => /^\d{4}$/.test(y))
  ).size;
  return { n, top, years };
});

// 图挂了（磁盘文件丢失/被清）不留红叉：换成占位符
const broken = ref(new Set());
const onImgErr = (id) => { broken.value = new Set(broken.value).add(id); };
const showImg = (h) => h.has_image && !broken.value.has(h.id);

async function load() {
  loading.value = true;
  try {
    const d = await api.honorList();
    list.value = d.list || [];
    err.value = '';
  } catch (e) {
    err.value = e.message || '加载失败';
  } finally {
    loading.value = false;
  }
}

onMounted(() => {
  mq = window.matchMedia('(prefers-reduced-motion: reduce)');
  reduced.value = mq.matches;
  mq.addEventListener?.('change', onMq);
  load();
  // 观察行容器实测宽（不要用 innerWidth 减 padding 去猜）：转屏/缩放后必须重算 k
  if (rowsRef.value) {
    ro = new ResizeObserver(([e]) => { vw.value = Math.round(e.contentRect.width) || vw.value; });
    ro.observe(rowsRef.value);
    vw.value = rowsRef.value.clientWidth || vw.value;
  }
});

onBeforeUnmount(() => {
  ro?.disconnect();
  mq?.removeEventListener?.('change', onMq);
});

// 列表空→有、以及 flow↔hall 切换都会让 rowsRef 卸载重挂。只要元素变了就重新绑定观察，
// 不能沿用旧的「!ro 才绑」——ro 一直在，但它观察的是已被卸掉的旧节点
watch(rowsRef, (el) => {
  if (el) {
    ro?.disconnect();
    ro = new ResizeObserver(([e]) => { vw.value = Math.round(e.contentRect.width) || vw.value; });
    ro.observe(el);
    vw.value = el.clientWidth || vw.value;
  }
});
</script>

<template>
  <div class="honor-page">
    <!-- 全站环境光：固定且裁剪，绝不参与页面滚动尺寸 -->
    <div class="honor-bg" aria-hidden="true">
      <span class="bg-orb bg-orb-a"></span>
      <span class="bg-orb bg-orb-b"></span>
    </div>

    <div class="honor-wrap">
      <!-- ============ 盛典 Hero ============ -->
      <section class="hero">
        <div class="hero-inner">
          <span class="hero-halo" aria-hidden="true"></span>
          <span class="hero-medal">
            <svg viewBox="0 0 24 24" aria-hidden="true">
              <defs>
                <linearGradient id="hg" x1="0" y1="0" x2="0" y2="1">
                  <stop offset="0" stop-color="#fbe48a" />
                  <stop offset=".5" stop-color="#e8b73e" />
                  <stop offset="1" stop-color="#b8860b" />
                </linearGradient>
              </defs>
              <path d="M6 3h12v4.6c0 2.6-2.4 4.7-5.3 4.95L12 12.6l-.7-.05C8.4 12.3 6 10.2 6 7.6Z" fill="url(#hg)" />
              <path d="M6.2 5H4.4a2.6 2.6 0 0 0 1.1 5" fill="none" stroke="url(#hg)" stroke-width="1.5" stroke-linecap="round" />
              <path d="M17.8 5h1.8a2.6 2.6 0 0 1-1.1 5" fill="none" stroke="url(#hg)" stroke-width="1.5" stroke-linecap="round" />
              <rect x="11" y="12.8" width="2" height="2.9" fill="url(#hg)" />
              <path d="M9.2 18h5.6l-.7-2.3H9.9Z" fill="url(#hg)" />
              <rect x="7.4" y="18" width="9.2" height="1.7" rx=".7" fill="url(#hg)" />
            </svg>
          </span>

          <!-- 星芒：减弱动态时不闪 -->
          <span class="hero-spark s1" aria-hidden="true">✦</span>
          <span class="hero-spark s2" aria-hidden="true">✦</span>
          <span class="hero-spark s3" aria-hidden="true">✦</span>
          <span class="hero-spark s4" aria-hidden="true">✧</span>

          <p class="hero-eyebrow">HALL&nbsp;&nbsp;OF&nbsp;&nbsp;HONOR</p>
          <h1 class="hero-title">荣誉墙</h1>
          <div class="hero-rule">
            <i></i><span class="hero-rule-gem">🏵</span><i></i>
          </div>
          <p class="hero-quote">每一块奖牌背后，都是一整个赛季的加班</p>

          <div class="hero-stats">
            <div class="hs-i">
              <b>{{ stat.n }}</b><span>荣誉总数</span>
            </div>
            <i class="hs-sep"></i>
            <div class="hs-i">
              <b>{{ stat.top }}</b><span>国家级 / 国际级</span>
            </div>
            <i class="hs-sep" v-if="stat.years"></i>
            <div class="hs-i" v-if="stat.years">
              <b>{{ stat.years }}</b><span>获奖年份</span>
            </div>
          </div>

          <button v-if="auth.user?.is_admin" class="hero-adm" type="button" @click="router.push('/admin-console')">
            ⚙️ 后台维护
          </button>
        </div>
      </section>

      <!-- 加载态保持高度：整块塌缩会让浏览器把 scrollY 钳回顶部（本仓 3753f7a 的教训） -->
      <div v-if="loading" class="card honor-skel" aria-busy="true">
        <span class="sk-bar" v-for="i in 3" :key="i"></span>
      </div>

      <div v-else-if="err" class="card honor-rstate err">
        <p>😵 {{ err }}</p>
        <button class="adm" type="button" @click="load">重试</button>
      </div>

      <div v-else-if="!list.length" class="card honor-rstate">
        <el-empty description="荣誉墙正在建设中" />
        <button v-if="auth.user?.is_admin" class="adm" type="button" @click="router.push('/admin-console')">
          🏅 去上传第一张奖状
        </button>
      </div>

      <template v-else>
        <!-- 视图切换 -->
        <div class="viewbar">
          <div class="vb-label">
            <i class="vb-ln"></i>
            <span class="vb-name">{{ mode === 'flow' ? '流动展厅' : '荣誉殿堂' }}</span>
            <i class="vb-ln"></i>
          </div>
          <div class="seg" role="tablist">
            <button type="button" role="tab" :aria-selected="mode === 'flow'" :class="{ on: mode === 'flow' }" @click="mode = 'flow'">
              🌊 流动展厅
            </button>
            <button type="button" role="tab" :aria-selected="mode === 'hall'" :class="{ on: mode === 'hall' }" @click="mode = 'hall'">
              🏛 荣誉殿堂
            </button>
          </div>
        </div>

        <!-- ===== ① 流动展厅：跑马灯（reduced 时同一套 DOM 退化成可横滚的静态行） ===== -->
        <div v-if="mode === 'flow'" class="honor-rows" ref="rowsRef" :class="{ 'is-reduced': reduced }" :style="trackVars">
          <div
            v-for="lane in lanes" :key="lane.i"
            class="marquee" :class="{ 'is-rev': lane.rev }"
          >
            <div class="track" :style="{ animationDuration: lane.dur + 's' }">
              <div class="half" v-for="c in copies" :key="c">
                <button
                  v-for="(h, ci) in lane.half" :key="`${h.id}-${c}-${ci}`"
                  class="hcard" type="button" draggable="false"
                  :class="[`t-${tierOf(h)}`, { 'is-static': !showImg(h) }]"
                  :aria-hidden="(c > 1 || ci >= lane.n) ? 'true' : null"
                  :tabindex="(c === 1 && ci < lane.n && showImg(h)) ? 0 : -1"
                  @click="showImg(h) && openImage(h.image_url, h.title)"
                >
                  <span class="haccent" aria-hidden="true"></span>
                  <span class="hpic">
                    <img
                      v-if="showImg(h)" :src="h.image_url" :alt="h.title"
                      decoding="async" draggable="false" @error="onImgErr(h.id)"
                    />
                    <span v-else class="hph">🏅</span>
                  </span>
                  <b class="httl">{{ h.title }}</b>
                  <span class="htier" v-if="h.award_level">
                    <i class="tdot" aria-hidden="true"></i>{{ h.award_level }}
                  </span>
                  <span class="hmeta" v-if="h.award_date">🗓 {{ h.award_date }}</span>
                  <span class="hwin" v-if="h.winner">👤 {{ h.winner }}</span>
                </button>
              </div>
            </div>
          </div>
          <p class="honor-tip" v-if="!reduced">把鼠标停在某一行可以暂停滚动 · 点奖状看大图</p>
        </div>

        <!-- ===== ② 荣誉殿堂：证书网格（更大、带描述；类名与跑马灯完全分开） ===== -->
        <div v-else class="hall-grid">
          <button
            v-for="h in list" :key="h.id"
            class="gcard" type="button" draggable="false"
            :class="[`t-${tierOf(h)}`, { 'is-static': !showImg(h) }]"
            @click="showImg(h) && openImage(h.image_url, h.title)"
          >
            <span class="haccent" aria-hidden="true"></span>
            <span class="gpic">
              <img
                v-if="showImg(h)" :src="h.image_url" :alt="h.title"
                loading="lazy" decoding="async" draggable="false" @error="onImgErr(h.id)"
              />
              <span v-else class="gph">🏅</span>
            </span>
            <b class="gttl">{{ h.title }}</b>
            <span class="htier" v-if="h.award_level">
              <i class="tdot" aria-hidden="true"></i>{{ h.award_level }}
            </span>
            <span class="gdesc" v-if="h.description">{{ h.description }}</span>
            <span class="gfoot">
              <span v-if="h.award_date">🗓 {{ h.award_date }}</span>
              <span v-if="h.winner">👤 {{ h.winner }}</span>
            </span>
          </button>
        </div>
      </template>
    </div>
  </div>
</template>

<style scoped lang="scss">
.honor-page { position: relative; min-height: 100vh; padding: 18px 14px 40px; }
.honor-wrap { position: relative; z-index: 1; max-width: 1200px; margin: 0 auto; }

// 全站环境光
.honor-bg { position: fixed; inset: 0; z-index: 0; overflow: hidden; pointer-events: none; }
.bg-orb { position: absolute; border-radius: 50%; filter: blur(90px); opacity: .5; }
.bg-orb-a {
  width: 460px; height: 460px; left: -120px; top: -80px;
  background: color-mix(in srgb, var(--primary) 22%, transparent);
}
.bg-orb-b {
  width: 520px; height: 520px; right: -160px; bottom: -140px;
  background: color-mix(in srgb, var(--primary) 16%, transparent);
}

.card {
  background: var(--card-bg);
  border: 1px solid var(--border);
  border-radius: 12px;
  padding: 18px 20px;
  margin-bottom: 14px;
}

// ==================== Hero ====================
.hero {
  position: relative; overflow: hidden;
  border-radius: 18px; margin-bottom: 18px;
  // 深色盛典底（浅主题下是海军蓝；幽灵模式覆盖成近黑+红），金色在两种底上都立得住
  --hero-a: #13265a;
  --hero-b: #0a1330;
  --hero-c: #080e22;
  background:
    radial-gradient(900px 380px at 50% -12%, color-mix(in srgb, var(--primary) 55%, transparent), transparent 72%),
    radial-gradient(560px 300px at 12% 118%, color-mix(in srgb, var(--primary) 32%, transparent), transparent 70%),
    radial-gradient(520px 280px at 90% 120%, rgba(212, 175, 55, .16), transparent 70%),
    linear-gradient(160deg, var(--hero-a), var(--hero-b) 62%, var(--hero-c));
  // 细网格纹 + 顶部高光
  &::before {

    content: ''; position: absolute; inset: 0;
    background-image:
      linear-gradient(rgba(255, 255, 255, .04) 1px, transparent 1px),
      linear-gradient(90deg, rgba(255, 255, 255, .04) 1px, transparent 1px);
    background-size: 34px 34px;
    mask-image: radial-gradient(700px 360px at 50% 0%, #000, transparent 75%);
  }
  &::after {
    content: ''; position: absolute; left: 0; right: 0; top: 0; height: 1px;
    background: linear-gradient(90deg, transparent, rgba(247, 221, 138, .65), transparent);
  }
}
.hero-inner {
  position: relative; z-index: 1;
  display: flex; flex-direction: column; align-items: center;
  padding: 38px 24px 32px;
  animation: hero-rise .7s cubic-bezier(.22, .61, .36, 1) both;
}
@keyframes hero-rise { from { opacity: 0; transform: translateY(16px); } to { opacity: 1; transform: none; } }

// 奖牌 + 旋转光环
.hero-medal {
  position: relative;
  width: 78px; height: 78px; margin-bottom: 14px;
  border-radius: 50%;
  padding: 2px;
  background: conic-gradient(from 0deg, #b8860b, #fbe48a, #b8860b, #fbe48a, #b8860b);
  box-shadow: 0 10px 30px -8px rgba(212, 175, 55, .55);
  svg {
    display: block; width: 100%; height: 100%; padding: 16px;
    border-radius: 50%;
    background: radial-gradient(circle at 32% 26%, #1d3268, #0b1430 75%);
  }
}
.hero-halo {
  position: absolute; top: 38px; width: 108px; height: 108px; border-radius: 50%;
  background: conic-gradient(from 0deg, transparent 0 70%, rgba(247, 221, 138, .55) 85%, transparent 100%);
  animation: hero-spin 9s linear infinite;
  pointer-events: none;
}
@keyframes hero-spin { to { transform: rotate(360deg); } }

.hero-spark {
  position: absolute; color: #f2d98a; pointer-events: none;
  animation: twinkle 2.8s ease-in-out infinite;
}
.s1 { top: 26px; left: 50%; margin-left: 72px; font-size: 15px; }
.s2 { top: 56px; left: 50%; margin-left: -96px; font-size: 11px; animation-delay: .7s; }
.s3 { top: 118px; left: 50%; margin-left: 132px; font-size: 10px; animation-delay: 1.3s; }
.s4 { top: 96px; left: 50%; margin-left: -140px; font-size: 13px; animation-delay: 1.9s; }
@keyframes twinkle { 0%, 100% { opacity: .15; transform: scale(.8); } 50% { opacity: 1; transform: scale(1.15); } }

.hero-eyebrow {
  margin: 0; font-size: 11px; letter-spacing: .5em; text-indent: .5em;
  color: rgba(247, 221, 138, .85);
}
.hero-title {
  margin: 8px 0 0; font-size: 46px; letter-spacing: 6px; text-indent: 6px;
  background: linear-gradient(180deg, #fff8e0, #f3d27a 55%, #d9a934);
  -webkit-background-clip: text; background-clip: text; color: transparent;
  text-shadow: 0 6px 30px rgba(212, 175, 55, .25);
}
.hero-rule { display: flex; align-items: center; gap: 10px; margin: 14px 0 0; width: 240px; max-width: 70%; }
.hero-rule i { flex: 1; height: 1px; background: linear-gradient(90deg, transparent, rgba(247, 221, 138, .7)); }
.hero-rule i:last-child { background: linear-gradient(90deg, rgba(247, 221, 138, .7), transparent); }
.hero-rule-gem { font-size: 15px; }
.hero-quote { margin: 14px 0 0; font-size: 13px; color: rgba(231, 238, 255, .78); }

.hero-stats {
  display: flex; align-items: stretch; gap: 22px;
  margin-top: 24px; padding: 16px 30px;
  border: 1px solid rgba(247, 221, 138, .22); border-radius: 14px;
  background: rgba(255, 255, 255, .045);
  backdrop-filter: blur(4px);
}
.hs-i { display: flex; flex-direction: column; align-items: center; gap: 4px; min-width: 78px; }
.hs-i b {
  font-size: 30px; line-height: 1.1;
  background: linear-gradient(180deg, #ffe9a8, #e3b341);
  -webkit-background-clip: text; background-clip: text; color: transparent;
}
.hs-i span { font-size: 11px; color: rgba(231, 238, 255, .72); text-align: center; }
.hs-sep { width: 1px; align-self: stretch; background: linear-gradient(180deg, transparent, rgba(247, 221, 138, .35), transparent); }

.hero-adm {
  margin-top: 22px; cursor: pointer;
  font-size: 13px; padding: 9px 22px; border-radius: 999px;
  color: #ffeec2; background: rgba(255, 255, 255, .08);
  border: 1px solid rgba(247, 221, 138, .45);
  transition: all .2s ease;
  &:hover { background: rgba(247, 221, 138, .18); transform: translateY(-1px); }
}

// ==================== 骨架 / 状态 ====================
.honor-skel { display: flex; flex-direction: column; gap: 14px; }
.sk-bar {
  height: 196px; border-radius: 10px; background: var(--surface-2);
  animation: sk-pulse 1.2s ease-in-out infinite;
}
@keyframes sk-pulse { 50% { opacity: .55; } }
.honor-rstate { text-align: center; }
.honor-rstate p { color: var(--text-2); }
.adm {
  font-size: 12px; padding: 5px 12px; border-radius: 8px; cursor: pointer;
  color: var(--primary); background: var(--primary-tint);
  border: 1px solid color-mix(in srgb, var(--primary) 33%, transparent);
  transition: all .2s ease;
  &:hover { background: color-mix(in srgb, var(--primary) 18%, transparent); }
}

// ==================== 视图切换 ====================
.viewbar {
  display: flex; align-items: center; justify-content: space-between;
  gap: 12px; flex-wrap: wrap; margin-bottom: 16px;
}
.vb-label { display: flex; align-items: center; gap: 12px; flex: 1; min-width: 180px; }
.vb-ln { flex: 1; height: 1px; background: linear-gradient(90deg, transparent, var(--border-2)); }
.vb-ln:last-child { background: linear-gradient(90deg, var(--border-2), transparent); }
.vb-name {
  font-size: 15px; font-weight: 600; letter-spacing: 3px; color: var(--text);
  white-space: nowrap;
}
.seg {
  display: inline-flex; padding: 3px; gap: 2px;
  background: var(--card-bg); border: 1px solid var(--border); border-radius: 999px;
  button {
    font: inherit; font-size: 12px; white-space: nowrap; cursor: pointer;
    padding: 6px 16px; border: 0; border-radius: 999px;
    color: var(--text-2); background: transparent; transition: all .2s ease;
    &:hover { color: var(--text); background: var(--surface-2); }
    &.on { color: #fff; background: var(--primary); font-weight: 600; }
  }
}

// ==================== 荣誉等级配色（金/银/铜） ====================
.t-gold   { --t: #b8860b; --t2: #ecc94b; }
.t-silver { --t: #64748b; --t2: #aab4c2; }
.t-bronze { --t: #a2602a; --t2: #cf8f5b; }

// 卡片共用：顶边色条 / 等级胶囊 / 奖牌点
.haccent {
  position: absolute; top: 0; left: 12px; right: 12px; height: 3px; border-radius: 0 0 3px 3px;
  background: linear-gradient(90deg, var(--t), var(--t2), var(--t));
  opacity: .85;
}
.htier {
  display: inline-flex; align-items: center; gap: 5px;
  align-self: flex-start; max-width: 100%;
  font-size: 11px; line-height: 1; padding: 4px 8px; border-radius: 6px;
  color: var(--t);
  background: color-mix(in srgb, var(--t) 12%, transparent);
  border: 1px solid color-mix(in srgb, var(--t) 26%, transparent);
  white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
}
.tdot {
  flex: none; width: 7px; height: 7px; border-radius: 50%;
  background: radial-gradient(circle at 30% 30%, var(--t2), var(--t));
}

// ==================== 跑马灯 ====================
.honor-rows { display: flex; flex-direction: column; gap: 14px; }

/* 行视口：必须 overflow:hidden，否则轨道会把整页撑出横向滚动 */
.marquee {
  overflow: hidden;
  padding: 2px 0;
  /* 两侧渐隐：把硬切边变成有意的观感（mask 不是颜色，双主题通吃） */
  -webkit-mask-image: linear-gradient(90deg, transparent, #000 4%, #000 96%, transparent);
  mask-image: linear-gradient(90deg, transparent, #000 4%, #000 96%, transparent);
  /* hover 与键盘聚焦都要能停下来（卡片是按钮，Tab 过去时不停根本点不到） */
  &:hover .track, &:focus-within .track { animation-play-state: paused; }
}

.track {
  display: flex;
  width: max-content;          /* 必须：否则轨道塌成视口宽，-50% 就没有意义了 */
  will-change: transform;
  animation-name: honor-rtl;
  animation-timing-function: linear;
  animation-iteration-count: infinite;
}
.marquee.is-rev .track { animation-name: honor-ltr; }

/* 两份等宽副本：位移 -50% 恰好等于一份的宽度，回绕点不可见 */
@keyframes honor-rtl { from { transform: translate3d(0, 0, 0); } to { transform: translate3d(-50%, 0, 0); } }
@keyframes honor-ltr { from { transform: translate3d(-50%, 0, 0); } to { transform: translate3d(0, 0, 0); } }

.half { display: flex; }

.hcard {
  position: relative;
  flex: none;
  width: var(--hw);
  margin-right: var(--hg);     /* ★ 间距做进卡片自身，不能用容器 gap（见文件头注释①） */
  /* 固定卡高 = 图盒 + 文本区最坏情况（padding 20 + 4×gap 24 + 标题 2 行 36.4 + 等级胶囊 19
     + 日期 13.2 + 获奖者 13.2）。不固定的话行与行会参差，一面墙基线全散。
     用 min-height 而不是 height：字体度量有偏差时卡片长高，而不是把文字挤出边框。 */
  min-height: calc(var(--bh) + 126px);
  display: flex; flex-direction: column; gap: 6px;
  padding: 10px; text-align: left; cursor: pointer;
  background: var(--card-bg);
  border: 1px solid var(--border);
  border-radius: 10px;
  font: inherit; color: inherit;
  transition: border-color .2s ease, transform .2s ease, box-shadow .2s ease;
  &:hover {
    border-color: color-mix(in srgb, var(--t) 45%, var(--border));
    transform: translateY(-3px);
    box-shadow: 0 14px 28px -14px color-mix(in srgb, var(--t) 60%, transparent);
    .hpic img { transform: scale(1.06); }
  }
  &.is-static { cursor: default; &:hover { transform: none; box-shadow: none; .hpic img { transform: none; } } }
}

.hpic {
  display: flex; align-items: center; justify-content: center;
  height: var(--bh); border-radius: 8px; overflow: hidden;
  background: var(--surface-2);
  box-shadow: inset 0 0 0 1px var(--border), inset 0 -10px 18px -14px rgba(0, 0, 0, .35);
  img {
    width: 100%; height: 100%; object-fit: contain;  /* contain 不裁奖状，细节交给全屏查看器 */
    transition: transform .25s ease;
  }
}
.hph { font-size: 34px; opacity: .5; }

.httl {
  font-size: 13px; line-height: 1.4; color: var(--text);
  display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden;
}
.hmeta, .hwin {
  font-size: 11px; color: var(--text-2);
  white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
}
.hwin { color: var(--text-3); }

.honor-tip { margin: 2px 0 0; text-align: center; font-size: 12px; color: var(--text-3); }

// ==================== 荣誉殿堂网格 ====================
.hall-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(236px, 1fr));
  gap: 18px;
}
.gcard {
  position: relative;
  display: flex; flex-direction: column; gap: 9px;
  padding: 14px 14px 12px; text-align: left; cursor: pointer;
  background: var(--card-bg);
  border: 1px solid var(--border); border-radius: 14px;
  font: inherit; color: inherit;
  transition: border-color .2s ease, transform .25s ease, box-shadow .25s ease;
  &:hover {
    border-color: color-mix(in srgb, var(--t) 45%, var(--border));
    transform: translateY(-4px);
    box-shadow: 0 20px 38px -18px color-mix(in srgb, var(--t) 65%, transparent);
    .gpic img { transform: scale(1.05); }
  }
  &.is-static { cursor: default; &:hover { transform: none; box-shadow: none; .gpic img { transform: none; } } }
}
.gpic {
  display: flex; align-items: center; justify-content: center;
  height: 168px; border-radius: 10px; overflow: hidden;
  background: var(--surface-2);
  box-shadow: inset 0 0 0 1px var(--border), inset 0 -12px 22px -16px rgba(0, 0, 0, .35);
  img { width: 100%; height: 100%; object-fit: contain; transition: transform .3s ease; }
}
.gph { font-size: 46px; opacity: .5; }
.gttl {
  font-size: 15px; line-height: 1.45; color: var(--text);
  display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden;
}
.gdesc {
  font-size: 12px; line-height: 1.55; color: var(--text-2);
  display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden;
}
.gfoot {
  margin-top: auto; padding-top: 4px;
  display: flex; align-items: center; justify-content: space-between; gap: 8px;
  font-size: 11px; color: var(--text-3);
  span { white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
}

/* 降级：不滚，但内容一张不少、可手动横滚（内容不可达才是最坏的降级） */
.honor-rows.is-reduced {
  .marquee { overflow-x: auto; scrollbar-width: thin; }
  .track { animation: none; }
}
@media (prefers-reduced-motion: reduce) {
  .track { animation: none !important; }
  .sk-bar { animation: none; }
  .hero-inner { animation: none; }
  .hero-halo { animation: none; }
  .hero-spark { animation: none; opacity: .5; }
}

@media (max-width: 768px) {
  .hero-inner { padding: 30px 16px 26px; }
  .hero-title { font-size: 36px; letter-spacing: 5px; text-indent: 5px; }
  .hero-stats { gap: 14px; padding: 14px 18px; }
  .hs-i { min-width: 64px; }
  .hs-i b { font-size: 24px; }
  .s2, .s4 { display: none; }
  .viewbar { justify-content: center; }
  .hall-grid { grid-template-columns: repeat(auto-fill, minmax(158px, 1fr)); gap: 12px; }
  .gpic { height: 132px; }
  .honor-tip { display: none; }
}
</style>

<!-- 非 scoped：幽灵模式下的变量覆盖。
     不能在 scoped 块里用 :global(html.ghost-mode) 嵌套 —— Sass 会把后代选择器吞掉、
     属性落到裸 html.ghost-mode 上，被 .hero/.t-* 自身的声明盖过（2026-09-26 实测）。
     全部用 .honor-page 前缀收窄，不外泄到别的页面。 -->
<style lang="scss">
html.ghost-mode .honor-page .hero {
  --hero-a: #1c0d10;
  --hero-b: #0c0708;
  --hero-c: #070506;
}
html.ghost-mode .honor-page .t-gold   { --t: #e6b93e; --t2: #f7dd8a; }
html.ghost-mode .honor-page .t-silver { --t: #9aa7b8; --t2: #c3ccda; }
html.ghost-mode .honor-page .t-bronze { --t: #d99a66; --t2: #efbd8f; }
</style>
