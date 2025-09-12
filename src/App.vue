<script setup>
import { ref, computed, onMounted } from 'vue'

/* -------------------------
   Data & state
-------------------------- */
const sads = ref([])   // Supraglottic Airway Devices
const etts = ref([])   // Endotracheal tubes

// Mode: 'sadToEtt' (default) or 'ettToSad'
const flow = ref('sadToEtt')

// Shared inputs
const clearance = ref(0.5) // mm

/* --- SAD → ETT selections --- */
const selectedSAD = ref(null)            // choose SAD first
const selectedETTSize = ref(null)        // then size
const selectedETTModel = ref(null)       // then model (optional)

/* --- ETT → SAD selections --- */
const ettSize_E2S = ref(null)            // ETT size (ID)
const ettModel_E2S = ref(null)           // ETT model (brand) optional
const useWorstCase_E2S = ref(true)       // conservative worst-case OD for size

onMounted(async () => {
  const [sadsRes, ettsRes] = await Promise.all([
    fetch('/sads.json'),
    fetch('/etts.json'),
  ])
  sads.value = await sadsRes.json()
  etts.value = await ettsRes.json()
})

/* -------------------------
   Helpers & computed
-------------------------- */

// Unique ETT sizes (IDs) available
const ettSizes = computed(() => {
  const sizes = new Set()
  for (const e of etts.value) {
    const id = Number(e.internal_mm)
    if (!Number.isNaN(id)) sizes.add(id)
  }
  return Array.from(sizes).sort((a,b)=>a-b)
})

// Models for a given ETT size
function modelsForSize(size) {
  const s = Number(size)
  return etts.value
    .filter(e => Number(e.internal_mm) === s && !Number.isNaN(Number(e.external_mm)))
    .sort((a,b)=>Number(a.external_mm) - Number(b.external_mm))
}

// Verdict core
function verdictFrom(sadID, ettOD, need) {
  const gap = sadID - ettOD
  if (gap < 0) return { state: 'no-fit', gap }
  if (gap < need) return { state: 'no-fit', gap }
  if (gap <= need + 0.5) return { state: 'tight', gap }
  return { state: 'fit', gap }
}

function labelState(s) {
  return s === 'fit' ? 'Fits' : s === 'tight' ? 'Tight' : s === 'no-fit' ? 'Doesn’t fit' : 'Unknown'
}

/* -------------------------
   SAD → ETT flow
-------------------------- */

// Size-level recommendations for chosen SAD (uses worst-case OD per size)
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
    const rank = { fit: 0, tight: 1, 'no-fit': 2, unknown: 3 }
    return (rank[a.state]-rank[b.state]) || (a.size - b.size)
  })
})

// Model-level recommendations once a size is chosen
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

// Final verdict for SAD → ETT (model if picked, else worst-case for size)
const finalVerdict_S2E = computed(() => {
  if (!selectedSAD.value) return null
  const sadID = Number(selectedSAD.value.internal_mm)
  const need = Number(clearance.value)

  if (selectedETTModel.value) {
    const od = Number(selectedETTModel.value.external_mm)
    const v = verdictFrom(sadID, od, need)
    return { ...v, sadID, od, need, mode: 'model' }
  }
  if (selectedETTSize.value != null) {
    const models = modelsForSize(selectedETTSize.value)
    if (!models.length) return null
    const worstOD = Math.max(...models.map(m => Number(m.external_mm)))
    const v = verdictFrom(sadID, worstOD, need)
    return { ...v, sadID, od: worstOD, need, mode: 'size-worst' }
  }
  return null
})

/* -------------------------
   ETT → SAD flow
-------------------------- */

// Models for the chosen size (E2S)
const models_E2S = computed(() => {
  if (ettSize_E2S.value == null) return []
  return modelsForSize(ettSize_E2S.value)
})

// Effective OD in ETT→SAD mode (worst-case or chosen model)
const effectiveETT_OD_E2S = computed(() => {
  if (ettSize_E2S.value == null) return null
  const models = models_E2S.value
  if (!models.length) return null
  if (useWorstCase_E2S.value) {
    return Math.max(...models.map(m => Number(m.external_mm)))
  }
  if (ettModel_E2S.value) {
    return Number(ettModel_E2S.value.external_mm)
  }
  return null
})

// SAD verdicts for the chosen ETT (for listing all SADs)
const sadRecommendations_E2S = computed(() => {
  const od = Number(effectiveETT_OD_E2S.value)
  const need = Number(clearance.value)
  if (Number.isNaN(od)) return []

  return sads.value
    .map(s => {
      const sadID = Number(s.internal_mm)
      if (Number.isNaN(sadID)) return { sad: s, state: 'unknown', gap: null }
      const { state, gap } = verdictFrom(sadID, od, need)
      return { sad: s, state, gap }
    })
    .sort((a,b) => {
      const rank = { fit: 0, tight: 1, 'no-fit': 2, unknown: 3 }
      return (rank[a.state]-rank[b.state]) || (Number(a.sad.internal_mm) - Number(b.sad.internal_mm))
    })
})

// Final maths line for ETT→SAD (if there’s an effective OD)
const mathsLine_E2S = computed(() => {
  const od = Number(effectiveETT_OD_E2S.value)
  const need = Number(clearance.value)
  if (Number.isNaN(od)) return ''
  return `Using ETT OD ${od.toFixed(2)} mm + clearance ${need.toFixed(2)} mm`
})
</script>

<template>
  <main class="container">
    <h1>Airway Compatibility</h1>

    <!-- Mode switch -->
    <div class="mode">
      <label><input type="radio" value="sadToEtt" v-model="flow"> SAD → ETT</label>
      <label><input type="radio" value="ettToSad" v-model="flow"> ETT → SAD</label>
    </div>

    <!-- Shared control -->
    <div class="field">
      <label>Clearance (mm): {{ clearance.toFixed(2) }}</label>
      <input type="range" min="0" max="1.5" step="0.1" v-model.number="clearance" />
      <small>Default 0.5 mm; adjust per device/preference.</small>
    </div>

    <!-- ===========================
         MODE: SAD → ETT
         =========================== -->
    <section v-if="flow==='sadToEtt'">
      <!-- 1) Choose SAD -->
      <div class="field">
        <label>SAD (Supraglottic Airway Device)</label>
        <select v-model="selectedSAD" @change="selectedETTSize=null; selectedETTModel=null">
          <option :value="null">— choose a SAD —</option>
          <option v-for="s in sads" :key="s.name + s.internal_mm" :value="s">
            {{ s.name }} <span v-if="s.manufacturer">— {{ s.manufacturer }}</span>
            — ID {{ Number(s.internal_mm).toFixed(2) }} mm
          </option>
        </select>
      </div>

      <!-- Context -->
      <div v-if="selectedSAD" class="context">
        <p><strong>Selected SAD:</strong> {{ selectedSAD.name }}
          <span v-if="selectedSAD.manufacturer">— {{ selectedSAD.manufacturer }}</span>
          — ID {{ Number(selectedSAD.internal_mm).toFixed(2) }} mm
        </p>
      </div>

      <!-- 2) Size recommendations (worst-case OD) -->
      <div v-if="selectedSAD" class="sizes">
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
      </div>

      <!-- 3) Model list within a size -->
      <div v-if="selectedSAD && selectedETTSize != null" class="models">
        <h3>Models for size {{ Number(selectedETTSize).toFixed(1) }}</h3>
        <div class="list">
          <label
            v-for="m in modelRecommendations"
            :key="m.model.name + m.od"
            class="row model"
            :class="m.state"
          >
            <input type="radio" name="model-s2e" :value="m.model" v-model="selectedETTModel" />
            <div class="model-info">
              <div class="model-name">
                {{ m.model.name }}
                <span v-if="m.model.manufacturer">— {{ m.model.manufacturer }}</span>
              </div>
              <div class="model-sub">
                OD {{ m.od.toFixed(2) }} mm ·
                <span class="badge" :data-state="m.state">{{ labelState(m.state) }}</span>
              </div>
            </div>
          </label>
        </div>
      </div>

      <!-- 4) Final verdict -->
      <section v-if="finalVerdict_S2E" class="result"
               :class="{ fit: finalVerdict_S2E.state==='fit', tight: finalVerdict_S2E.state==='tight', nofit: finalVerdict_S2E.state==='no-fit' }">
        <div class="badge big" :data-state="finalVerdict_S2E.state">{{ labelState(finalVerdict_S2E.state) }}</div>
        <p class="details">
          Using {{ finalVerdict_S2E.mode==='model' ? 'selected model OD' : 'worst-case OD for this size' }}:
          {{ finalVerdict_S2E.od.toFixed(2) }} + {{ finalVerdict_S2E.need.toFixed(2) }} ≤ {{ finalVerdict_S2E.sadID.toFixed(2) }}
        </p>
        <p class="details" v-if="selectedETTModel?.notes">ETT notes: {{ selectedETTModel.notes }}</p>
        <p class="details" v-if="selectedSAD?.notes">SAD notes: {{ selectedSAD.notes }}</p>
      </section>
    </section>

    <!-- ===========================
         MODE: ETT → SAD
         =========================== -->
    <section v-else>
      <!-- 1) Choose ETT size & (optionally) model -->
      <div class="grid">
        <div class="field">
          <label>ETT size (ID, mm)</label>
          <select v-model="ettSize_E2S" @change="ettModel_E2S=null">
            <option :value="null">— choose size —</option>
            <option v-for="s in ettSizes" :key="s" :value="s">{{ s.toFixed(1) }}</option>
          </select>
          <small v-if="!ettSizes.length">No ETT sizes found in data.</small>
        </div>

        <div class="field">
          <label>ETT model (brand)</label>
          <select v-model="ettModel_E2S" :disabled="useWorstCase_E2S || !models_E2S.length">
            <option :value="null">— choose a model —</option>
            <option v-for="m in models_E2S" :key="m.name + m.external_mm" :value="m">
              {{ m.name }} <span v-if="m.manufacturer">— {{ m.manufacturer }}</span>
              — OD {{ Number(m.external_mm).toFixed(2) }} mm
            </option>
          </select>
          <label class="row" style="margin-top:.4rem;">
            <input type="checkbox" v-model="useWorstCase_E2S" />
            <span style="margin-left:.5rem;">Use worst-case OD for this size (conservative)</span>
          </label>
        </div>
      </div>

      <!-- 2) List SADs with verdicts for the chosen ETT -->
      <div v-if="ettSize_E2S != null && effectiveETT_OD_E2S != null" class="models">
        <h3>Compatible SADs</h3>
        <p class="maths">{{ mathsLine_E2S }}</p>

        <div class="list">
          <div
            v-for="rec in sadRecommendations_E2S"
            :key="rec.sad.name + rec.sad.internal_mm"
            class="row model"
            :class="rec.state"
          >
            <div class="model-info">
              <div class="model-name">
                {{ rec.sad.name }} <span v-if="rec.sad.manufacturer">— {{ rec.sad.manufacturer }}</span>
              </div>
              <div class="model-sub">
                SAD ID {{ Number(rec.sad.internal_mm).toFixed(2) }} mm ·
                <span class="badge" :data-state="rec.state">{{ labelState(rec.state) }}</span>
              </div>
            </div>
          </div>
        </div>
        <p class="details" v-if="ettModel_E2S?.notes">ETT notes: {{ ettModel_E2S.notes }}</p>
      </div>
    </section>
  </main>
</template>

<style>
:root { font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif; }
.container { max-width: 1100px; margin: 2rem auto; padding: 1rem; }
h1 { font-size: 1.6rem; margin-bottom: 0.75rem; }
h2 { margin: .5rem 0; font-size: 1.2rem; }
h3 { margin: .5rem 0; font-size: 1.05rem; }

.mode { display: flex; justify-content: center; gap: 1rem; margin: 1rem 0; margin-bottom: .75rem; }
.mode label { display: flex; align-items: center; gap: .4rem; font-weight: 600; }

.field { margin-bottom: 0.8rem; }
.field label { display: block; font-weight: 600; margin-bottom: .35rem; }
select, input[type="range"] { width: 100%; }
small { color: #666; }
.context { margin: .4rem 0 .8rem; }

.grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 1rem; }
@media (max-width: 900px) { .grid { grid-template-columns: 1fr; } }

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

.details { margin: 0.25rem 0; }
.maths   { margin: 0.25rem 0; font-family: ui-monospace, SFMono-Regular, Menlo, monospace; color: #333; }
</style>
