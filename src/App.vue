<template>
  <main class="container">
    <header class="header">
      <h1>Airway Device Compatibility Checker</h1>
      <p class="disclaimer">
        estimation only, manufacturer data, check specs, always check actual device fit before clinical use.
      </p>
      <p v-if="errorMsg" class="error" role="alert">{{ errorMsg }}</p>
    </header>

    <!-- Mode toggle -->
    <section class="mode">
      <fieldset class="fieldset">
        <legend class="legend">Mode</legend>
        <div class="segmented">
          <label :class="['chip', flow==='sadToEtt' && 'active']">
            <input type="radio" name="flow" value="sadToEtt" v-model="flow" />
            SAD → ETT
          </label>
          <label :class="['chip', flow==='ettToSad' && 'active']">
            <input type="radio" name="flow" value="ettToSad" v-model="flow" />
            ETT → SAD
          </label>
        </div>
      </fieldset>
    </section>

    <!-- Clearance control -->
    <section class="controls">
      <fieldset class="fieldset">
        <legend class="legend">Clearance (mm)</legend>
        <div class="clearance">
          <input
            type="range"
            min="0"
            max="1.5"
            step="0.1"
            v-model.number="clearance"
            :aria-valuetext="clearance.toFixed(1) + ' millimetres'"
          />
          <input
            class="number"
            type="number"
            min="0"
            max="1.5"
            step="0.1"
            v-model.number="clearance"
          />
          <span class="unit">mm</span>
        </div>
        <p class="hint">We’ll prefer a “fit” when ETT OD ≤ SAD ID − clearance. “Tight” if ≤ SAD ID. Otherwise “no-fit”.</p>
      </fieldset>
    </section>

    <!-- Flow: SAD to ETT -->
    <section v-if="flow==='sadToEtt'" class="panel">
      <h2>SAD → ETT</h2>
      <div class="grid">
        <div class="card">
          <fieldset class="fieldset">
            <legend class="legend">Choose a SAD</legend>

            <label class="label">SAD size</label>
            <select v-model="sadSize">
              <option value="">— Select size —</option>
              <option v-for="size in sadSizes" :key="size" :value="size">
                {{ size }}
              </option>
            </select>

            <label class="label">SAD model (optional)</label>
            <select v-model="sadModelIdx" :disabled="!sadSize">
              <option :value="''">— Any model —</option>
              <option v-for="(m, i) in sadModelsForSize" :key="i" :value="String(i)">
                {{ m.name }} <span v-if="m.manufacturer">— {{ nice(m.manufacturer) }}</span>
                (ID: {{ fmtmm(m.internal_mm) }} mm)
              </option>
            </select>
          </fieldset>
        </div>

        <div class="card">
          <fieldset class="fieldset">
            <legend class="legend">Choose an ETT</legend>

            <label class="label">ETT internal size (mm)</label>
            <select v-model="ettSize">
              <option value="">— Select I.D. —</option>
              <option v-for="sz in ettIDs" :key="sz" :value="String(sz)">{{ sz }}</option>
            </select>

            <label class="label">ETT model (optional)</label>
            <select v-model="ettModelIdx" :disabled="!ettSize">
              <option :value="''">— Any model —</option>
              <option v-for="(m, i) in ettModelsForID" :key="i" :value="String(i)">
                {{ m.name }} <span v-if="m.manufacturer">— {{ nice(m.manufacturer) }}</span>
                (OD: {{ fmtmm(m.external_mm) }} mm)
              </option>
            </select>
          </fieldset>
        </div>
      </div>

      <!-- Results -->
      <div class="results" aria-live="polite">
        <!-- Neutral estimate when only SAD size chosen -->
        <div v-if="sadSize && !sadModel && !ettModel" class="result neutral">
          <div class="badge big" data-state="estimate">Estimate</div>
          <p>
            With a size <strong>{{ sadSize }}</strong> SAD, the <em>worst-case</em> ETT outer diameter
            to aim for is
            <strong>{{ fmtmm(estimateMaxOD) }} mm</strong>
            (SAD ID − clearance).
          </p>
          <p class="muted">
            Pick a specific SAD and/or ETT model to see a colour-coded verdict.
          </p>
        </div>

        <!-- Detailed verdict when a model is chosen -->
        <div v-if="sadModel && ettModel" class="result" :class="verdict.state">
          <div class="badge big" :data-state="verdict.state">{{ labelState(verdict.state) }}</div>
          <div class="calcbox">
            <div class="row">
              <span class="key">SAD</span>
              <span class="val">
                {{ sadModel.name }} <span v-if="sadModel.manufacturer">— {{ nice(sadModel.manufacturer) }}</span>
                (ID: {{ fmtmm(sadModel.internal_mm) }} mm)
              </span>
            </div>
            <div class="row">
              <span class="key">ETT</span>
              <span class="val">
                {{ ettModel.name }} <span v-if="ettModel.manufacturer">— {{ nice(ettModel.manufacturer) }}</span>
                (OD: {{ fmtmm(ettModel.external_mm) }} mm)
              </span>
            </div>
            <div class="row">
              <span class="key">Clearance</span>
              <span class="val">{{ fmtmm(clearance) }} mm</span>
            </div>
            <div class="row">
              <span class="key">Test</span>
              <span class="val monospace">
                {{ fmtmm(ettModel.external_mm) }} ≤ {{ fmtmm(sadModel.internal_mm) }} − {{ fmtmm(clearance) }}
                → <strong>{{ labelState(verdict.state) }}</strong>
              </span>
            </div>
          </div>
          <p v-if="verdict.state==='tight'" class="muted">
            Tight fit: OD ≤ ID but &gt; (ID − clearance). Consider stepping down or increasing clearance.
          </p>
          <p v-if="verdict.state==='no-fit'" class="muted">
            No fit with current clearance. Choose a smaller OD ETT or a larger ID SAD.
          </p>
        </div>
      </div>
    </section>

    <!-- Flow: ETT to SAD -->
    <section v-else class="panel">
      <h2>ETT → SAD</h2>
      <div class="grid">
        <div class="card">
          <fieldset class="fieldset">
            <legend class="legend">Choose an ETT</legend>

            <label class="label">ETT internal size (mm)</label>
            <select v-model="ettSize">
              <option value="">— Select I.D. —</option>
              <option v-for="sz in ettIDs" :key="sz" :value="String(sz)">{{ sz }}</option>
            </select>

            <label class="label">ETT model (optional)</label>
            <select v-model="ettModelIdx" :disabled="!ettSize">
              <option :value="''">— Any model —</option>
              <option v-for="(m, i) in ettModelsForID" :key="i" :value="String(i)">
                {{ m.name }} <span v-if="m.manufacturer">— {{ nice(m.manufacturer) }}</span>
                (OD: {{ fmtmm(m.external_mm) }} mm)
              </option>
            </select>
          </fieldset>
        </div>

        <div class="card" aria-live="polite">
          <h3 class="subtle">Compatible SADs</h3>
          <div v-if="!ettModel" class="muted">Pick a specific ETT model to list SAD compatibility.</div>
          <ul v-else class="list">
            <li v-for="rec in sadRecommendations" :key="rec.key" :class="['item', rec.state]">
              <span class="badge" :data-state="rec.state">{{ labelState(rec.state) }}</span>
              <span class="name">
                {{ rec.sad.name }} <span v-if="rec.sad.manufacturer">— {{ nice(rec.sad.manufacturer) }}</span>
              </span>
              <span class="meta">ID: {{ fmtmm(rec.sad.internal_mm) }} mm</span>
            </li>
          </ul>
        </div>
      </div>
    </section>

    <footer class="footer">
      <small>© {{ new Date().getFullYear() }} — For education only. Not a substitute for clinical judgement.</small>
    </footer>
  </main>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue'

/** ---------- State ---------- */
const sads = ref([])
const etts = ref([])
const errorMsg = ref('')

const flow = ref(localStorage.getItem('flow') || 'sadToEtt')
const clearance = ref(clamp(+localStorage.getItem('clearance') || 0.5, 0, 1.5))

watch(flow, v => localStorage.setItem('flow', v))
watch(clearance, v => localStorage.setItem('clearance', String(clamp(v, 0, 1.5))))

// Selections
const sadSize = ref('')
const sadModelIdx = ref('')
const ettSize = ref('')
const ettModelIdx = ref('')

/** ---------- Load data ---------- */
onMounted(async () => {
  try {
    const [sadsRes, ettsRes] = await Promise.all([
      fetch('/sads.json'),
      fetch('/etts.json'),
    ])
    if (!sadsRes.ok || !ettsRes.ok) throw new Error('Failed to load data files')
    sads.value = await sadsRes.json()
    etts.value = await ettsRes.json()
  } catch (err) {
    console.error(err)
    errorMsg.value = 'Could not load device data. Please refresh or check the deployment paths.'
    sads.value = []
    etts.value = []
  }
})

/** ---------- Helpers ---------- */
const VERDICT_RANK = Object.freeze({ fit: 0, tight: 1, 'no-fit': 2, unknown: 3 })

function nice (str) {
  return typeof str === 'string' ? str.replace(/\s+/g, ' ').trim() : str
}
function fmtmm (n) {
  if (n == null || Number.isNaN(+n)) return '—'
  return Number(n).toFixed(1)
}
function clamp (v, min, max) {
  return Math.min(max, Math.max(min, Number.isFinite(+v) ? +v : min))
}
function labelState (s) {
  switch (s) {
    case 'fit': return 'Fit'
    case 'tight': return 'Tight'
    case 'no-fit': return 'No fit'
    default: return '—'
  }
}

/** ---------- Derived sets ---------- */
// Unique ETT I.D. sizes present in dataset
const ettIDs = computed(() => {
  const ids = new Set(etts.value.map(e => Number(e.internal_mm)).filter(n => Number.isFinite(n)))
  return Array.from(ids).sort((a, b) => a - b).map(n => Number(n.toFixed(1)))
})

// SAD sizes as present in their names (simple extract: first size token)
const sadSizes = computed(() => {
  // Prefer explicit “size X” token in name; fall back to integer part before comma
  const sizes = new Set()
  for (const s of sads.value) {
    const name = String(s.name || '')
    const m = name.match(/\bsize\s*([0-9.]+)/i) || name.match(/\b([0-9.]+)\b/)
    if (m) sizes.add(m[1])
  }
  return Array.from(sizes).sort((a, b) => Number(a) - Number(b))
})

const sadModelsForSize = computed(() => {
  if (!sadSize.value) return []
  const size = String(sadSize.value).trim()
  return sads.value
    .filter(s => String(s.name).toLowerCase().includes(`size ${size}`.toLowerCase()) || String(s.name).includes(size))
    .sort((a, b) => Number(a.internal_mm) - Number(b.internal_mm))
})

const ettModelsForID = computed(() => {
  if (!ettSize.value) return []
  const id = Number(ettSize.value)
  return etts.value
    .filter(e => Number(e.internal_mm) === id)
    .sort((a, b) => Number(a.external_mm) - Number(b.external_mm))
})

const sadModel = computed(() => {
  if (sadModelIdx.value === '' || !sadModelsForSize.value.length) return null
  const i = Number(sadModelIdx.value)
  return sadModelsForSize.value[i] || null
})

const ettModel = computed(() => {
  if (ettModelIdx.value === '' || !ettModelsForID.value.length) return null
  const i = Number(ettModelIdx.value)
  return ettModelsForID.value[i] || null
})

/** ---------- Calculations ---------- */
const estimateMaxOD = computed(() => {
  // When only SAD size chosen (no specific model), we show worst-case:
  // take the smallest ID among models for that size, subtract clearance
  if (!sadModelsForSize.value.length) return null
  const minID = Math.min(...sadModelsForSize.value.map(m => Number(m.internal_mm)).filter(Number.isFinite))
  const est = minID - Number(clearance.value)
  return est > 0 ? Number(est.toFixed(1)) : 0
})

function stateFor (sadID, ettOD, clear) {
  if (!Number.isFinite(sadID) || !Number.isFinite(ettOD)) return 'unknown'
  const target = sadID - clear
  if (ettOD <= target) return 'fit'
  if (ettOD <= sadID) return 'tight'
  return 'no-fit'
}

const verdict = computed(() => {
  if (!sadModel.value || !ettModel.value) return { state: 'unknown' }
  const sadID = Number(sadModel.value.internal_mm)
  const ettOD = Number(ettModel.value.external_mm)
  const state = stateFor(sadID, ettOD, Number(clearance.value))
  return { state }
})

const sadRecommendations = computed(() => {
  if (!ettModel.value) return []
  const ettOD = Number(ettModel.value.external_mm)
  const clear = Number(clearance.value)
  return sads.value
    .map((sad, idx) => {
      const sadID = Number(sad.internal_mm)
      const state = stateFor(sadID, ettOD, clear)
      return {
        key: `${idx}-${sadID}`,
        sad,
        state,
      }
    })
    .sort((a, b) =>
      (VERDICT_RANK[a.state] - VERDICT_RANK[b.state]) ||
      (Number(a.sad.internal_mm) - Number(b.sad.internal_mm))
    )
})
</script>

<style scoped>
/* Layout */
.container {
  max-width: 1000px;
  margin: 0 auto;
  padding: 2rem 1rem 4rem;
  font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Ubuntu, Cantarell, Noto Sans, Arial, "Apple Color Emoji", "Segoe UI Emoji";
  color: #1f2937;
}
.header { text-align: center; margin-bottom: 1.25rem; }
h1 { font-size: 1.6rem; margin: 0 0 .25rem; }
h2 { font-size: 1.25rem; margin: 1rem 0 .5rem; }
h3.subtle { margin: 0 0 .5rem; font-weight: 600; color: #374151; }
.disclaimer {
  font-size: .9rem; color: #6b7280;
}
.footer { margin-top: 2rem; text-align: center; color: #6b7280; }

.error {
  margin: .75rem auto 0;
  max-width: 720px;
  background: #fdecea;
  border: 1px solid #e74c3c;
  color: #9b2c2c;
  padding: .75rem;
  border-radius: 10px;
}

/* Controls */
.mode { display: flex; justify-content: center; margin: 1rem 0; }
.segmented {
  display: inline-flex; gap: .5rem; background: #f3f4f6; padding: .25rem; border-radius: 999px;
  border: 1px solid #e5e7eb;
}
.chip {
  display: inline-flex; align-items: center; gap: .5rem;
  padding: .4rem .9rem; border-radius: 999px; cursor: pointer; user-select: none;
  font-weight: 600; color: #374151;
}
.chip input { display: none; }
.chip.active { background: white; box-shadow: 0 1px 2px rgba(0,0,0,.06); border: 1px solid #e5e7eb; }

.controls { display: flex; justify-content: center; margin: 1rem 0; }
.fieldset { border: 1px solid #e5e7eb; border-radius: 12px; padding: 1rem; }
.legend { font-weight: 700; padding: 0 .25rem; }
.label { display: block; font-weight: 600; margin-top: .75rem; margin-bottom: .25rem; }
select, input[type="number"], input[type="range"] {
  width: 100%;
  box-sizing: border-box;
}
select, input[type="number"] {
  border: 1px solid #e5e7eb; border-radius: 10px; padding: .5rem .6rem; background: #fff;
}
input[type="number"] { width: 7rem; }

.clearance { display: flex; align-items: center; gap: .75rem; }
.clearance .number { width: 6rem; }
.unit { color: #6b7280; }
.hint { margin: .5rem 0 0; color: #6b7280; font-size: .9rem; }

/* Panels */
.panel { margin-top: .5rem; }
.grid {
  display: grid; gap: 1rem;
  grid-template-columns: repeat(2, minmax(0, 1fr));
}
@media (max-width: 860px) {
  .grid { grid-template-columns: 1fr; }
}
.card {
  background: white; border: 1px solid #e5e7eb; border-radius: 14px; padding: 1rem;
  box-shadow: 0 1px 2px rgba(0,0,0,.03);
}

/* Results */
.results { margin-top: 1rem; }
.result {
  border: 1px solid #e5e7eb; border-radius: 14px; padding: 1rem; background: #fff;
}
.result.fit { border-color: #16a34a33; background: #16a34a0d; }
.result.tight { border-color: #f59e0b33; background: #f59e0b0d; }
.result.no-fit, .result.no-fit { border-color: #ef444433; background: #ef44440d; }

.badge {
  display: inline-block; font-weight: 700; font-size: .75rem; padding: .25rem .5rem; border-radius: 999px;
  border: 1px solid #e5e7eb; background: #fff; margin-right: .5rem;
}
.badge.big { font-size: .9rem; padding: .35rem .65rem; }
.badge[data-state="fit"] { border-color: #16a34a; color: #166534; }
.badge[data-state="tight"] { border-color: #f59e0b; color: #92400e; }
.badge[data-state="no-fit"] { border-color: #ef4444; color: #991b1b; }
.badge[data-state="estimate"] { border-color: #64748b; color: #334155; }

.calcbox {
  margin-top: .75rem; border: 1px dashed #cbd5e1; border-radius: 12px; padding: .75rem; background: #f8fafc;
}
.row { display: grid; grid-template-columns: 120px 1fr; gap: .5rem; margin: .25rem 0; }
.key { color: #64748b; font-weight: 600; }
.val { color: #111827; }
.monospace { font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", monospace; }

.muted { color: #6b7280; font-size: .95rem; }

/* List */
.list { list-style: none; padding: 0; margin: 0; display: grid; gap: .5rem; }
.item {
  display: grid; align-items: center;
  grid-template-columns: auto 1fr auto; gap: .5rem;
  padding: .5rem .6rem; border: 1px solid #e5e7eb; border-radius: 12px; background: #fff;
}
.name { font-weight: 600; }
.meta { color: #6b7280; font-variant-numeric: tabular-nums; }
.item.fit { border-color: #16a34a66; background: #16a34a0a; }
.item.tight { border-color: #f59e0b66; background: #f59e0b0a; }
.item.no-fit { border-color: #ef444466; background: #ef44440a; }
</style>
