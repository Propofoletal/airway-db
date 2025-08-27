<script setup>
import { ref, computed, onMounted } from 'vue'

const sgas = ref([])
const etts = ref([])

const selectedETTSize = ref(null)   // ETT size by internal diameter (ID)
const selectedETTModel = ref(null)  // specific brand/model (drives OD)
const useWorstCase = ref(true)      // use largest OD among models for that size (conservative)

const selectedSGA = ref(null)
const clearance = ref(0.5)          // mm

onMounted(async () => {
  const [sgasRes, ettsRes] = await Promise.all([
    fetch('/sgas.json'),
    fetch('/etts.json'),
  ])
  sgas.value = await sgasRes.json()
  etts.value = await ettsRes.json()
})

/** Distinct ETT size options (by internal_mm) */
const ettSizes = computed(() => {
  const sizes = new Set()
  for (const e of etts.value) {
    const id = Number(e.internal_mm)
    if (!Number.isNaN(id)) sizes.add(id)
  }
  return Array.from(sizes).sort((a,b)=>a-b)
})

/** Models available for the chosen ETT size */
const modelsForSelectedSize = computed(() => {
  if (selectedETTSize.value == null) return []
  const size = Number(selectedETTSize.value)
  return etts.value
    .filter(e => Number(e.internal_mm) === size && !Number.isNaN(Number(e.external_mm)))
    .sort((a,b)=>Number(a.external_mm) - Number(b.external_mm))
})

/** Effective OD used in the check (worst case or chosen model) */
const effectiveETT_OD = computed(() => {
  if (!selectedETTSize.value) return null
  const models = modelsForSelectedSize.value
  if (!models.length) return null

  if (useWorstCase.value) {
    return Math.max(...models.map(m => Number(m.external_mm)))
  }
  if (selectedETTModel.value) {
    return Number(selectedETTModel.value.external_mm)
  }
  return null
})

/** Human-readable context for ETT selection */
const ettContextText = computed(() => {
  if (!selectedETTSize.value || effectiveETT_OD.value == null) return ''
  if (useWorstCase.value) {
    return `ETT size ${Number(selectedETTSize.value).toFixed(1)} (worst-case OD ${effectiveETT_OD.value.toFixed(2)} mm)`
  }
  if (selectedETTModel.value) {
    const m = selectedETTModel.value
    const cuff = Number(m.cuff_mm)
    const cuffTxt = !Number.isNaN(cuff) ? `; cuff ${cuff.toFixed(1)} mm` : ''
    return `${m.name} — size ${Number(m.internal_mm).toFixed(1)}; OD ${Number(m.external_mm).toFixed(2)} mm${cuffTxt}`
  }
  return ''
})

/** Compatibility verdict */
const verdict = computed(() => {
  if (!selectedSGA.value || effectiveETT_OD.value == null) return null

  const sgaID = Number(selectedSGA.value.internal_mm)   // SGA internal diameter
  const ettOD = Number(effectiveETT_OD.value)           // ETT external diameter
  const need = Number(clearance.value)

  if (Number.isNaN(sgaID) || Number.isNaN(ettOD)) {
    return { state: 'unknown', msg: 'Missing diameter data.' }
  }

  const gap = sgaID - ettOD
  if (gap < 0) {
    return { state: 'no-fit', msg: `ETT OD ${ettOD.toFixed(2)} > SGA ID ${sgaID.toFixed(2)}` }
  }
  if (gap < need) {
    return { state: 'no-fit', msg: `Needs ≥ ${need.toFixed(2)} mm clearance; gap = ${gap.toFixed(2)} mm` }
  }
  // “tight” if within +0.5 mm beyond the clearance
  if (gap <= need + 0.5) {
    return { state: 'tight', msg: `Gap ${gap.toFixed(2)} mm (tight margin)` }
  }
  return { state: 'fit', msg: `Gap ${gap.toFixed(2)} mm` }
})

function stateLabel(s) {
  return s === 'fit' ? 'Fits' :
         s === 'tight' ? 'Tight' :
         s === 'no-fit' ? 'Doesn’t fit' : 'Unknown'
}

const mathsLine = computed(() => {
  if (!selectedSGA.value || effectiveETT_OD.value == null) return ''
  const id = Number(selectedSGA.value.internal_mm)
  const od = Number(effectiveETT_OD.value)
  const need = Number(clearance.value)
  return `${od.toFixed(2)} + ${need.toFixed(2)} ≤ ${id.toFixed(2)}`
})
</script>

<template>
  <main class="container">
    <h1>ETT ↔︎ SGA Compatibility</h1>

    <section class="grid">
      <!-- ETT SIZE -->
      <div class="field">
        <label>ETT size (ID, mm)</label>
        <select v-model="selectedETTSize" @change="selectedETTModel = null">
          <option :value="null">— choose size —</option>
          <option v-for="s in ettSizes" :key="s" :value="s">{{ s.toFixed(1) }}</option>
        </select>
        <small v-if="!ettSizes.length">No ETT sizes found in data.</small>
      </div>

      <!-- ETT MODEL -->
      <div class="field">
        <label>ETT model (brand)</label>
        <select v-model="selectedETTModel" :disabled="useWorstCase || !modelsForSelectedSize.length">
          <option :value="null">— choose a model —</option>
          <option v-for="m in modelsForSelectedSize" :key="m.name + m.external_mm" :value="m">
            {{ m.name }} — OD {{ Number(m.external_mm).toFixed(2) }} mm
          </option>
        </select>
        <label class="row" style="margin-top:.4rem;">
          <input type="checkbox" v-model="useWorstCase" />
          <span style="margin-left:.5rem;">Use worst-case OD for this size (conservative)</span>
        </label>
      </div>

      <!-- SGA -->
      <div class="field">
        <label>SGA</label>
        <select v-model="selectedSGA">
          <option :value="null">— choose an SGA —</option>
          <option v-for="s in sgas" :key="s.name + s.internal_mm" :value="s">
            {{ s.name }} — ID {{ Number(s.internal_mm).toFixed(2) }} mm
          </option>
        </select>
      </div>

      <!-- Clearance -->
      <div class="field">
        <label>Clearance (mm): {{ clearance.toFixed(2) }}</label>
        <input type="range" min="0" max="1.5" step="0.1" v-model.number="clearance" />
        <small>Default 0.5 mm. Adjust to device/preference.</small>
      </div>
    </section>

    <!-- Context -->
    <section v-if="selectedETTSize || selectedSGA" class="context">
      <p v-if="ettContextText"><strong>ETT:</strong> {{ ettContextText }}</p>
      <p v-if="selectedSGA"><strong>SGA:</strong> {{ selectedSGA.name }} — ID {{ Number(selectedSGA.internal_mm).toFixed(2) }} mm</p>
    </section>

    <!-- Verdict -->
    <section v-if="verdict" class="result"
             :class="{
               fit: verdict.state==='fit',
               tight: verdict.state==='tight',
               nofit: verdict.state==='no-fit',
               unknown: verdict.state==='unknown'
             }">
      <div class="badge" :data-state="verdict.state">
        {{ stateLabel(verdict.state) }}
      </div>
      <p class="details">{{ verdict.msg }}</p>
      <p class="maths">{{ mathsLine }}</p>

      <p v-if="!useWorstCase && selectedETTModel?.notes" class="notes">ETT notes: {{ selectedETTModel.notes }}</p>
      <p v-if="selectedSGA?.notes" class="notes">SGA notes: {{ selectedSGA.notes }}</p>
    </section>
  </main>
</template>

<style>
:root { font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif; }
.container { max-width: 1000px; margin: 2rem auto; padding: 1rem; }
h1 { font-size: 1.6rem; margin-bottom: 1rem; }

.grid { display: grid; grid-template-columns: repeat(4, 1fr); gap: 1rem; }
@media (max-width: 1000px) { .grid { grid-template-columns: 1fr; } }

.field label { display: block; font-weight: 600; margin-bottom: 0.4rem; }
.row { display: flex; align-items: center; }
select, input[type="range"] { width: 100%; }
small { color: #666; }

.context { margin-top: .5rem; padding: .5rem 0; color: #333; }

.result { margin-top: 1rem; padding: 1rem; border: 2px solid transparent; border-radius: 10px; }
.result.fit    { background: #e9f7ef; border-color: #2ecc71; }
.result.tight  { background: #fff8e6; border-color: #f1c40f; }
.result.nofit  { background: #fdecea; border-color: #e74c3c; }
.result.unknown{ background: #f2f2f2; border-color: #bdc3c7; }

.badge {
  display: inline-block; font-weight: 700; margin-bottom: 0.4rem;
  padding: 0.25rem 0.6rem; border-radius: 999px; border: 1px solid rgba(0,0,0,0.07);
}
.badge[data-state="fit"] { background: #2ecc71; color: white; }
.badge[data-state="tight"] { background: #f1c40f; color: #111; }
.badge[data-state="no-fit"] { background: #e74c3c; color: white; }

.details { margin: 0.25rem 0; }
.maths   { margin: 0.25rem 0; font-family: ui-monospace, SFMono-Regular, Menlo, monospace; color: #333; }
.notes   { margin-top: 0.4rem; font-size: 0.9rem; color: #333; }
</style>
