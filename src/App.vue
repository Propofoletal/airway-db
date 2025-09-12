<script setup>
import { ref, computed, onMounted } from 'vue'

const sads = ref([])   // Supraglottic Airway Devices
const etts = ref([])   // Endotracheal tubes (with internal_mm and external_mm)

const selectedSAD = ref(null)
const clearance = ref(0.5) // mm default

// Selection after SAD
const selectedETTSize = ref(null)   // choose an ETT size (ID)
const selectedETTModel = ref(null)  // then (optionally) choose a specific model

onMounted(async () => {
  const [sadsRes, ettsRes] = await Promise.all([
    fetch('/sads.json'),
    fetch('/etts.json'),
  ])
  sads.value = await sadsRes.json()
  etts.value = await ettsRes.json()
})

/** Distinct ETT size options (by internal_mm) present in data */
const ettSizes = computed(() => {
  const sizes = new Set()
  for (const e of etts.value) {
    const id = Number(e.internal_mm)
    if (!Number.isNaN(id)) sizes.add(id)
  }
  return Array.from(sizes).sort((a,b)=>a-b)
})

/** Models for a given size (helper) */
function modelsForSize(size) {
  const s = Number(size)
  return etts.value
    .filter(e => Number(e.internal_mm) === s && !Number.isNaN(Number(e.external_mm)))
    .sort((a,b)=>Number(a.external_mm) - Number(b.external_mm))
}

/** Verdict helper */
function verdictFrom(sadID, ettOD, need) {
  const gap = sadID - ettOD
  if (gap < 0) return { state: 'no-fit', gap }
  if (gap < need) return { state: 'no-fit', gap }
  if (gap <= need + 0.5) return { state: 'tight', gap }
  return { state: 'fit', gap }
}

/** Size-level recommendations (use worst-case OD for each size) */
const sizeRecommendations = computed(() => {
  if (!selectedSAD.value) return []
  const sadID = Number(selectedSAD.value.internal_mm)
  const need = Number(clearance.value)
  if (Number.isNaN(sadID)) return []

  return ettSizes.value.map(size => {
    const models = modelsForSize(size)
    if (!models.length) return { size, state: 'unknown', worstOD: null, gap: null }
    const worstOD = Math.max(...models.map(m => Number(m.external_mm)))
    const { state, gap } = verdictFrom(sadID, worstOD, need)
    return { size, state, worstOD, gap }
  }).sort((a,b) => {
    // order: fit → tight → no-fit → unknown, then by size
    const rank = { fit: 0, tight: 1, 'no-fit': 2, unknown: 3 }
    return (rank[a.state]-rank[b.state]) || (a.size - b.size)
  })
})

/** Models and their verdicts for the currently selected size */
const modelRecommendations = computed(() => {
  if (!selectedSAD.value || selectedETTSize.value == null) return []
  const sadID = Number(selectedSAD.value.internal_mm)
  const need = Number(clearance.value)
  const models = modelsForSize(selectedETTSize.value)
  return models.map(m => {
    const od = Number(m.external_mm)
    const { state, gap } = verdictFrom(sadID, od, need)
    return { model: m, od, state, gap }
  }).sort((a,b) => {
    const rank = { fit: 0, tight: 1, 'no-fit': 2 }
    return (rank[a.state]-rank[b.state]) || (a.od - b.od)
  })
})

/** Final detailed verdict once a model is chosen */
const finalVerdict = computed(() => {
  if (!selectedSAD.value) return null
  if (selectedETTModel.value) {
    const sadID = Number(selectedSAD.value.internal_mm)
    const od = Number(selectedETTModel.value.external_mm)
    const need = Number(clearance.value)
    const v = verdictFrom(sadID, od, need)
    return { ...v, sadID, od, need, mode: 'model' }
  }
  if (selectedETTSize.value != null) {
    // fallback to worst-case of that size
    const sadID = Number(selectedSAD.value.internal_mm)
    const need = Number(clearance.value)
    const models = modelsForSize(selectedETTSize.value)
    if (!models.length) return null
    const worstOD = Math.max(...models.map(m => Number(m.external_mm)))
    const v = verdictFrom(sadID, worstOD, need)
    return { ...v, sadID, od: worstOD, need, mode: 'size-worst' }
  }
  return null
})

function labelState(s) {
  return s === 'fit' ? 'Fits' : s === 'tight' ? 'Tight' : s === 'no-fit' ? 'Doesn’t fit' : 'Unknown'
}
</script>

<template>
  <main class="container">
    <h1>SAD → ETT Compatibility</h1>

    <!-- 1) Choose SAD first -->
    <section class="grid">
      <div class="field">
        <label>SAD (Supraglottic Airway Device)</label>
        <select v-model="selectedSAD" @change="selectedETTSize=null; selectedETTModel=null">
          <option :value="null">— choose a SAD —</option>
          <option v-for="s in sads" :key="s.name + s.internal_mm" :value="s">
            {{ s.name }} <span v-if="s.manufacturer">— {{ s.manufacturer }}</span> — ID {{ Number(s.internal_mm).toFixed(2) }} mm
          </option>
        </select>
      </div>

      <div class="field">
        <label>Clearance (mm): {{ clearance.toFixed(2) }}</label>
        <input type="range" min="0" max="1.5" step="0.1" v-model.number="clearance" />
        <small>Default 0.5 mm; adjust per device/preference.</small>
      </div>
    </section>

    <!-- Context for SAD -->
    <section v-if="selectedSAD" class="context">
      <p><strong>Selected SAD:</strong> {{ selectedSAD.name }} <span v-if="selectedSAD.manufacturer">— {{ selectedSAD.manufacturer }}</span> — ID {{ Number(selectedSAD.internal_mm).toFixed(2) }} mm</p>
    </section>

    <!-- 2) Size-level recommendations (worst-case per size) -->
    <section v-if="selectedSAD" class="sizes">
      <h2>Recommended ETT sizes (by ID)</h2>
      <div class="chips">
        <button
          v-for="rec in sizeRecommendations"
          :key="rec.size"
          class="chip"
          :class="rec.state"
          @click="selectedETTSize = rec.size; selectedETTModel = null;"
          :title="rec.worstOD ? `Worst-case OD ${rec.worstOD.toFixed(2)} mm` : ''"
        >
          {{ rec.size.toFixed(1) }}
        </button>
      </div>
      <small>Colours use <em>worst-case OD</em> for each size across brands.</small>
    </section>

    <!-- 3) Within a size, pick a model (brand) -->
    <section v-if="selectedSAD && selectedETTSize != null" class="models">
      <h3>Models for size {{ Number(selectedETTSize).toFixed(1) }}</h3>
      <div class="list">
        <label
          v-for="m in modelRecommendations"
          :key="m.model.name + m.od"
          class="row model"
          :class="m.state"
        >
          <input type="radio" name="model" :value="m.model" v-model="selectedETTModel" />
          <div class="model-info">
            <div class="model-name">
              {{ m.model.name }} <span v-if="m.model.manufacturer">— {{ m.model.manufacturer }}</span>
            </div>
            <div class="model-sub">
              OD {{ m.od.toFixed(2) }} mm · <span class="badge" :data-state="m.state">{{ labelState(m.state) }}</span>
            </div>
          </div>
        </label>
      </div>
    </section>

    <!-- 4) Final verdict (chosen model or worst-case for the size) -->
    <section v-if="finalVerdict" class="result"
             :class="{ fit: finalVerdict.state==='fit', tight: finalVerdict.state==='tight', nofit: finalVerdict.state==='no-fit' }">
      <div class="badge big" :data-state="finalVerdict.state">{{ labelState(finalVerdict.state) }}</div>
      <p class="details">
        Using {{ finalVerdict.mode==='model' ? 'selected model OD' : 'worst-case OD for this size' }}:
        {{ finalVerdict.od.toFixed(2) }} + {{ finalVerdict.need.toFixed(2) }} ≤ {{ finalVerdict.sadID.toFixed(2) }}
      </p>
      <p class="details" v-if="selectedETTModel?.notes">ETT notes: {{ selectedETTModel.notes }}</p>
      <p class="details" v-if="selectedSAD?.notes">SAD notes: {{ selectedSAD.notes }}</p>
    </section>
  </main>
</template>

<style>
:root { font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif; }
.container { max-width: 1000px; margin: 2rem auto; padding: 1rem; }
h1 { font-size: 1.6rem; margin-bottom: 1rem; }
h2 { margin: .5rem 0; font-size: 1.2rem; }
h3 { margin: .5rem 0; font-size: 1.05rem; }

.grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 1rem; }
@media (max-width: 900px) { .grid { grid-template-columns: 1fr; } }

.field label { display: block; font-weight: 600; margin-bottom: .4rem; }
select, input[type="range"] { width: 100%; }
small { color: #666; }
.context { margin: .5rem 0 1rem; }

.sizes .chips { display: flex; flex-wrap: wrap; gap: .5rem; margin: .5rem 0; }
.chip {
  padding: .35rem .6rem; border-radius: 999px; border: 1px solid transparent; cursor: pointer;
  background: #f2f2f2;
}
.chip.fit   { background: #e9f7ef; border-color: #2ecc71; }
.chip.tight { background: #fff8e6; border-color: #f1c40f; }
.chip.no-fit{ background: #fdecea; border-color: #e74c3c; }

.models .list { display: grid; gap: .4rem; margin-top: .5rem; }
.row.model {
  display: flex; align-items: center; gap: .6rem; padding: .5rem; border-radius: 8px; border: 1px solid transparent;
}
.row.model.fit   { background: #e9f7ef; border-color: #2ecc71; }
.row.model.tight { background: #fff8e6; border-color: #f1c40f; }
.row.model.no-fit{ background: #fdecea; border-color: #e74c3c; }

.model-info { display: flex; flex-direction: column; }
.model-name { font-weight: 600; }
.model-sub { font-size: .9rem; color: #333; }

.result { margin-top: 1rem; padding: 1rem; border: 2px solid transparent; border-radius: 10px; }
.result.fit    { background: #e9f7ef; border-color: #2ecc71; }
.result.tight  { background: #fff8e6; border-color: #f1c40f; }
.result.nofit  { background: #fdecea; border-color: #e74c3c; }

.badge {
  display: inline-block; font-weight: 700; padding: .15rem .5rem; border-radius: 999px; border: 1px solid rgba(0,0,0,0.06);
}
.badge.big { padding: .25rem .7rem; margin-bottom: .4rem; }
.badge[data-state="fit"] { background: #2ecc71; color: white; }
.badge[data-state="tight"] { background: #f1c40f; color: #111; }
.badge[data-state="no-fit"] { background: #e74c3c; color: white; }
</style>
