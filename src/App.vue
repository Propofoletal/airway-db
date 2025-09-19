<template>
  <main class="container" :data-theme="effectiveTheme">
    <header class="header">
      <div class="toprow">
        <h1>Airway Device Compatibility Checker</h1>

        <!-- Theme switcher -->
        <label class="theme-switch">
          <span class="sr-only">Theme</span>
          <select v-model="themeChoice" title="Theme">
            <option value="auto">Auto</option>
            <option value="light">Light</option>
            <option value="dark">Dark</option>
          </select>
        </label>
      </div>

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
        <p class="hint">We prefer a “fit” when ETT OD ≤ SAD ID − clearance. “Tight” if ≤ SAD ID. Otherwise “no-fit”.</p>
      </fieldset>
    </section>

    <!-- Flow: SAD to ETT -->
    <section v-if="flow==='sadToEtt'" class="panel">
      <h2>SAD → ETT</h2>
      <div class="grid">
        <!-- SAD chooser (two-tier) -->
        <div class="card">
          <fieldset class="fieldset">
            <legend class="legend">Choose a SAD</legend>

            <!-- Tier 1: Type + Manufacturer (comma separator, i-gel®) -->
            <label class="label">SAD type</label>
            <select v-model="sadTypeKey">
              <option value="">— Select type —</option>
              <option v-for="t in sadTypes" :key="t" :value="t">{{ t }}</option>
            </select>

            <!-- Tier 2: Size -->
            <label class="label">SAD size</label>
            <select v-model="sadSize" :disabled="!sadTypeKey">
              <option value="">— Select size —</option>
              <option v-for="size in sadSizesForType" :key="size" :value="size">{{ size }}</option>
            </select>
          </fieldset>
        </div>

        <!-- ETT chooser (worst-case + custom OD and auto-fill) -->
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

            <label class="label">Custom ETT OD (mm, optional)</label>
            <div class="od-row">
              <input
                class="number"
                type="number"
                min="2"
                max="15"
                step="0.1"
                v-model.number="customEttOD"
                @input="customTouched = true"
                placeholder="Auto-fills from model or worst-case"
              />
              <button v-if="customTouched" class="btn-reset" type="button" @click="resetCustomOD">Reset</button>
            </div>
            <p class="hint">
              Auto-fills from the selected model, or worst-case for the chosen I.D. You can override by typing.
            </p>
          </fieldset>
        </div>
      </div>

      <!-- Results -->
      <div class="results" aria-live="polite">
        <!-- Neutral estimate when only SAD chosen (no ETT OD source yet) -->
        <div v-if="sadSize && !hasEttOD" class="result neutral">
          <div class="badge big" data-state="estimate">Estimate</div>
          <p>
            With <strong>{{ sadTypeKey }}</strong> size <strong>{{ sadSize }}</strong>,
            the <em>worst-case</em> ETT outer diameter to aim for is
            <strong>{{ fmtmm(estimateMaxOD) }} mm</strong> (min SAD ID for that size − clearance).
          </p>
          <p class="muted">
            Select an ETT model, enter a custom OD, or choose an I.D. for worst-case OD.
          </p>
        </div>

        <!-- Verdict when SAD size + some ETT OD source is available -->
        <div v-if="sadSize && hasEttOD" class="result" :class="verdict.state">
          <div class="badge big" :data-state="verdict.state">{{ labelState(verdict.state) }}</div>
          <div class="calcbox">
            <div class="row">
              <span class="key">SAD (family)</span>
              <span class="val">
                {{ sadTypeKey }}, size {{ sadSize }}
                <span class="muted">(using worst-case ID)</span>
              </span>
            </div>
            <div class="row">
              <span class="key">SAD ID used</span>
              <span class="val">{{ fmtmm(sadWorstCaseID) }} mm</span>
            </div>
            <div class="row">
              <span class="key">ETT OD used</span>
              <span class="val">
                {{ fmtmm(ettODUsed) }} mm
                <span class="muted">({{ ettODSourceLabel }})</span>
              </span>
            </div>
            <div class="row">
              <span class="key">Clearance</span>
              <span class="val">{{ fmtmm(clearance) }} mm</span>
            </div>
            <div class="row">
              <span class="key">Test</span>
              <span class="val monospace">
                {{ fmtmm(ettODUsed) }} ≤ {{ fmtmm(sadWorstCaseID) }} − {{ fmtmm(clearance) }}
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

            <label class="label">Custom ETT OD (mm, optional)</label>
            <div class="od-row">
              <input
                class="number"
                type="number"
                min="2"
                max="15"
                step="0.1"
                v-model.number="customEttOD"
                @input="customTouched = true"
                placeholder="Auto-fills from model or worst-case"
              />
              <button v-if="customTouched" class="btn-reset" type="button" @click="resetCustomOD">Reset</button>
            </div>
            <p class="hint">
              No model? We’ll list SADs using your custom OD, or worst-case OD for the chosen I.D. if custom is empty.
            </p>
          </fieldset>
        </div>

        <div class="card" aria-live="polite">
          <h3 class="subtle">Compatible SADs</h3>
          <div v-if="!hasEttOD" class="muted">Pick an ETT model, enter a custom OD, or choose an I.D. to use worst-case OD.</div>
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

// Theme (Auto/Light/Dark) with system follow
const themeChoice = ref(localStorage.getItem('themeChoice') || 'auto') // 'auto' | 'light' | 'dark'
const prefersDark = ref(false)
const effectiveTheme = computed(() => themeChoice.value === 'auto' ? (prefersDark.value ? 'dark' : 'light') : themeChoice.value)
watch(themeChoice, v => localStorage.setItem('themeChoice', v))

onMounted(() => {
  const mq = window.matchMedia('(prefers-color-scheme: dark)')
  const apply = () => { prefersDark.value = mq.matches }
  apply()
  mq.addEventListener ? mq.addEventListener('change', apply) : mq.addListener(apply)
})

/** SAD (two-tier) */
const sadTypeKey = ref('') // e.g. "i-gel®, Intersurgical"
const sadSize = ref('')     // e.g. "3"

/** ETT */
const ettSize = ref('')
const ettModelIdx = ref('')

// OD entry + behaviour
const customEttOD = ref(null)     // number | null
const customTouched = ref(false)  // user has typed in OD input
const resetCustomOD = () => { customEttOD.value = null; customTouched.value = false }

/** ---------- Load data ---------- */
onMounted(async () => {
  try {
    const [sadsRes, ettsRes] = await Promise.all([ fetch('/sads.json'), fetch('/etts.json') ])
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
function nice (str) { return typeof str === 'string' ? str.replace(/\s+/g, ' ').trim() : str }
function fmtmm (n) { if (n == null || Number.isNaN(+n)) return '—'; return Number(n).toFixed(1) }
function clamp (v, min, max) { return Math.min(max, Math.max(min, Number.isFinite(+v) ? +v : min)) }
function labelState (s) { return s==='fit'?'Fit':s==='tight'?'Tight':s==='no-fit'?'No fit':'—' }

// Normalise type names; ensure "i-gel®"; manufacturer fix; comma separator
// Normalise type names; strip ® and ™; compress spaces
function baseType (name) {
  const first = String(name || '').split(',')[0].trim()
  return first.replace(/[®™]/g, '').replace(/\s+/g, ' ')
}
function normManu (m) { return nice(String(m || '').replace(/Intersurgcial/i, 'Intersurgical')) }

/** ---------- SAD Grouping (Type + Manufacturer) ---------- */
const sadGroups = computed(() => {
  const groups = {}
  for (const s of sads.value) {
    const key = `${baseType(s.name)}, ${normManu(s.manufacturer)}`
    if (!groups[key]) groups[key] = []
    groups[key].push(s)
  }
  return groups
})
const sadTypes = computed(() => Object.keys(sadGroups.value).sort((a, b) => a.localeCompare(b)))
watch(sadTypeKey, () => { sadSize.value = '' })

/** Sizes available for selected Type/Manufacturer */
const sadSizesForType = computed(() => {
  if (!sadTypeKey.value) return []
  const group = sadGroups.value[sadTypeKey.value] || []
  const sizes = new Set()
  for (const s of group) {
    const m = String(s.name).match(/\bsize\s*([0-9.]+)/i)
    if (m) sizes.add(m[1])
  }
  return Array.from(sizes).sort((a, b) => Number(a) - Number(b))
})

/** Worst-case (smallest) ID for selected Type/Manufacturer + Size */
const sadWorstCaseID = computed(() => {
  if (!sadTypeKey.value || !sadSize.value) return null
  const group = sadGroups.value[sadTypeKey.value] || []
  const size = String(sadSize.value).trim().toLowerCase()
  const ids = group
    .filter(s => String(s.name).toLowerCase().includes(`size ${size}`))
    .map(s => Number(s.internal_mm))
    .filter(Number.isFinite)
  if (!ids.length) return null
  return Math.min(...ids)
})

/** ---------- ETT Derived sets + OD logic ---------- */
const ettIDs = computed(() => {
  const ids = new Set(etts.value.map(e => Number(e.internal_mm)).filter(n => Number.isFinite(n)))
  return Array.from(ids).sort((a, b) => a - b).map(n => Number(n.toFixed(1)))
})
const ettModelsForID = computed(() => {
  if (!ettSize.value) return []
  const id = Number(ettSize.value)
  return etts.value
    .filter(e => Number(e.internal_mm) === id)
    .sort((a, b) => Number(a.external_mm) - Number(b.external_mm))
})
const ettModel = computed(() => {
  if (ettModelIdx.value === '' || !ettModelsForID.value.length) return null
  const i = Number(ettModelIdx.value)
  return ettModelsForID.value[i] || null
})
const worstCaseEttOD = computed(() => {
  if (!ettSize.value) return null
  const ods = ettModelsForID.value.map(m => Number(m.external_mm)).filter(Number.isFinite)
  return ods.length ? Math.max(...ods) : null
})
const hasCustom = computed(() =>
  customTouched.value &&
  customEttOD.value !== null &&
  customEttOD.value !== '' &&
  Number.isFinite(Number(customEttOD.value))
)
const ettODUsed = computed(() => {
  if (hasCustom.value) return Number(customEttOD.value)
  if (ettModel.value && Number.isFinite(Number(ettModel.value.external_mm))) return Number(ettModel.value.external_mm)
  if (Number.isFinite(Number(worstCaseEttOD.value))) return Number(worstCaseEttOD.value)
  return null
})
const ettODSourceLabel = computed(() => {
  if (hasCustom.value) return 'custom'
  if (ettModel.value) return 'selected model'
  if (worstCaseEttOD.value != null && ettSize.value) return 'worst-case (database)'
  return '—'
})
const hasEttOD = computed(() => ettODUsed.value != null && Number.isFinite(Number(ettODUsed.value)))

/** Auto-fill behaviour (don’t overwrite typed values) */
watch(ettModelIdx, () => {
  if (!ettModel.value) return
  if (!customTouched.value) customEttOD.value = Number(ettModel.value.external_mm)
})
watch(ettSize, () => {
  if (ettModel.value) return
  if (!customTouched.value) customEttOD.value = worstCaseEttOD.value != null ? Number(worstCaseEttOD.value) : null
})
watch(customEttOD, (v) => {
  if (v === null || v === '') customTouched.value = false
})
watch(flow, () => { resetCustomOD() })

/** ---------- Calculations ---------- */
const estimateMaxOD = computed(() => {
  if (!sadWorstCaseID.value && sadWorstCaseID.value !== 0) return null
  const est = Number(sadWorstCaseID.value) - Number(clearance.value)
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
  if (!sadWorstCaseID.value && sadWorstCaseID.value !== 0) return { state: 'unknown' }
  if (!hasEttOD.value) return { state: 'unknown' }
  const sadID = Number(sadWorstCaseID.value)
  const ettOD = Number(ettODUsed.value)
  const state = stateFor(sadID, ettOD, Number(clearance.value))
  return { state }
})
const sadRecommendations = computed(() => {
  if (!hasEttOD.value) return []
  const ettOD = Number(ettODUsed.value)
  const clear = Number(clearance.value)
  return sads.value
    .map((sad, idx) => {
      const sadID = Number(sad.internal_mm)
      const state = stateFor(sadID, ettOD, clear)
      return { key: `${idx}-${sadID}`, sad, state }
    })
    .sort((a, b) =>
      (VERDICT_RANK[a.state] - VERDICT_RANK[b.state]) ||
      (Number(a.sad.internal_mm) - Number(b.sad.internal_mm))
    )
})
</script>

<style scoped>
/* ---------- Design tokens via CSS variables ---------- */
:root, .container[data-theme="light"] {
  --bg: #ffffff;
  --text: #1f2937;
  --muted: #6b7280;
  --card: #ffffff;
  --border: #e5e7eb;
  --shadow: 0 1px 2px rgba(0,0,0,.06);

  --accent-fit: #16a34a;
  --accent-tight: #f59e0b;
  --accent-nofit: #ef4444;
  --accent-est: #64748b;

  --tint-fit: rgba(22,163,74,0.05);
  --tint-tight: rgba(245,158,11,0.06);
  --tint-nofit: rgba(239,68,68,0.06);
}

@media (prefers-color-scheme: dark) {
  :root {
    --bg: #0b1020;
    --text: #e6e8ee;
    --muted: #a3a7b5;
    --card: #12172a;
    --border: #2a2f45;
    --shadow: 0 1px 2px rgba(0,0,0,.4);

    --tint-fit: rgba(22,163,74,0.12);
    --tint-tight: rgba(245,158,11,0.14);
    --tint-nofit: rgba(239,68,68,0.14);
  }
}

/* Force dark regardless of system when user picks "dark" */
.container[data-theme="dark"] {
  --bg: #0b1020;
  --text: #e6e8ee;
  --muted: #a3a7b5;
  --card: #12172a;
  --border: #2a2f45;
  --shadow: 0 1px 2px rgba(0,0,0,.4);

  --tint-fit: rgba(22,163,74,0.12);
  --tint-tight: rgba(245,158,11,0.14);
  --tint-nofit: rgba(239,68,68,0.14);
}

/* Force light when user picks "light" */
.container[data-theme="light"] {
  --bg: #ffffff;
  --text: #1f2937;
  --muted: #6b7280;
  --card: #ffffff;
  --border: #e5e7eb;
  --shadow: 0 1px 2px rgba(0,0,0,.06);

  --tint-fit: rgba(22,163,74,0.05);
  --tint-tight: rgba(245,158,11,0.06);
  --tint-nofit: rgba(239,68,68,0.06);
}

/* ---------- Base layout ---------- */
.container {
  max-width: 1000px;
  margin: 0 auto;
  padding: 2rem 1rem 4rem;
  font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Ubuntu, Cantarell, Noto Sans, Arial, "Apple Color Emoji", "Segoe UI Emoji";
  color: var(--text);
  background: var(--bg);
}
.header { margin-bottom: 1.25rem; }
.toprow {
  display: flex; align-items: center; justify-content: space-between; gap: 1rem;
}
h1 { font-size: 1.6rem; margin: 0 0 .25rem; }
h2 { font-size: 1.25rem; margin: 1rem 0 .5rem; }
h3.subtle { margin: 0 0 .5rem; font-weight: 600; color: var(--text); opacity: .9; }
.disclaimer { font-size: .9rem; color: var(--muted); }
.footer { margin-top: 2rem; text-align: center; color: var(--muted); }

/* Theme switcher */
.theme-switch select {
  border: 1px solid var(--border);
  background: var(--card);
  color: var(--text);
  border-radius: 10px;
  padding: .4rem .6rem;
}
.sr-only { position: absolute; width:1px; height:1px; overflow:hidden; clip:rect(0 0 0 0); white-space:nowrap; }

/* Alerts */
.error {
  margin: .75rem 0 0;
  background: color-mix(in oklab, var(--accent-nofit) 15%, transparent);
  border: 1px solid var(--accent-nofit);
  color: var(--text);
  padding: .75rem;
  border-radius: 10px;
}

/* Controls */
.mode { display: flex; justify-content: center; margin: 1rem 0; }
.segmented {
  display: inline-flex; gap: .5rem; background: var(--card); padding: .25rem; border-radius: 999px;
  border: 1px solid var(--border); box-shadow: var(--shadow);
}
.chip {
  display: inline-flex; align-items: center; gap: .5rem;
  padding: .4rem .9rem; border-radius: 999px; cursor: pointer; user-select: none;
  font-weight: 600; color: var(--text);
}
.chip input { display: none; }
.chip.active { background: var(--bg); box-shadow: var(--shadow); border: 1px solid var(--border); }

.controls { display: flex; justify-content: center; margin: 1rem 0; }
.fieldset { border: 1px solid var(--border); border-radius: 12px; padding: 1rem; background: var(--card); }
.legend { font-weight: 700; padding: 0 .25rem; }
.label { display: block; font-weight: 600; margin-top: .75rem; margin-bottom: .25rem; }
select, input[type="number"], input[type="range"] {
  width: 100%;
  box-sizing: border-box;
}
select, input[type="number"] {
  border: 1px solid var(--border); border-radius: 10px; padding: .5rem .6rem; background: var(--bg); color: var(--text);
}
input[type="number"] { width: 7rem; }
input::placeholder { color: color-mix(in oklab, var(--muted) 70%, transparent); }

.clearance { display: flex; align-items: center; gap: .75rem; }
.clearance .number { width: 6rem; }
.unit { color: var(--muted); }
.hint { margin: .5rem 0 0; color: var(--muted); font-size: .9rem; }

/* Panels */
.panel { margin-top: .5rem; }
.grid { display: grid; gap: 1rem; grid-template-columns: repeat(2, minmax(0, 1fr)); }
@media (max-width: 860px) { .grid { grid-template-columns: 1fr; } }
.card { background: var(--card); border: 1px solid var(--border); border-radius: 14px; padding: 1rem; box-shadow: var(--shadow); }

/* Results */
.results { margin-top: 1rem; }
.result { border: 1px solid var(--border); border-radius: 14px; padding: 1rem; background: var(--card); }
.result.fit { border-color: color-mix(in oklab, var(--accent-fit) 55%, var(--border)); background: var(--tint-fit); }
.result.tight { border-color: color-mix(in oklab, var(--accent-tight) 55%, var(--border)); background: var(--tint-tight); }
.result.no-fit { border-color: color-mix(in oklab, var(--accent-nofit) 55%, var(--border)); background: var(--tint-nofit); }

.badge {
  display: inline-block; font-weight: 700; font-size: .75rem; padding: .25rem .5rem; border-radius: 999px;
  border: 1px solid var(--border); background: var(--bg); margin-right: .5rem; color: var(--text);
}
.badge.big { font-size: .9rem; padding: .35rem .65rem; }
.badge[data-state="fit"] { border-color: var(--accent-fit); color: var(--accent-fit); }
.badge[data-state="tight"] { border-color: var(--accent-tight); color: var(--accent-tight); }
.badge[data-state="no-fit"] { border-color: var(--accent-nofit); color: var(--accent-nofit); }
.badge[data-state="estimate"] { border-color: var(--accent-est); color: var(--accent-est); }

.calcbox { margin-top: .75rem; border: 1px dashed var(--border); border-radius: 12px; padding: .75rem; background: color-mix(in oklab, var(--bg) 70%, var(--card)); }
.row { display: grid; grid-template-columns: 160px 1fr; gap: .5rem; margin: .25rem 0; }
.key { color: var(--muted); font-weight: 600; }
.val { color: var(--text); }
.monospace { font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", monospace; }

/* List */
.list { list-style: none; padding: 0; margin: 0; display: grid; gap: .5rem; }
.item {
  display: grid; align-items: center;
  grid-template-columns: auto 1fr auto; gap: .5rem;
  padding: .5rem .6rem; border: 1px solid var(--border); border-radius: 12px; background: var(--card);
}
.name { font-weight: 600; }
.meta { color: var(--muted); font-variant-numeric: tabular-nums; }
.item.fit { border-color: color-mix(in oklab, var(--accent-fit) 50%, var(--border)); background: var(--tint-fit); }
.item.tight { border-color: color-mix(in oklab, var(--accent-tight) 50%, var(--border)); background: var(--tint-tight); }
.item.no-fit { border-color: color-mix(in oklab, var(--accent-nofit) 50%, var(--border)); background: var(--tint-nofit); }

/* Small UI bits */
.od-row { display: flex; align-items: center; gap: .5rem; }
.btn-reset {
  border: 1px solid var(--border);
  background: var(--bg);
  color: var(--text);
  border-radius: 8px;
  padding: .45rem .6rem;
  cursor: pointer;
}
.btn-reset:hover { filter: brightness(1.05); }
</style>
