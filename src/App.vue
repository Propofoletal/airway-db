<script setup>
import { ref, computed, onMounted, watch } from "vue";

/* Data */
const sads = ref([]);   // [{ name, manufacturer, size:"Size 4, ...", internal_mm }]
const etts = ref([]);   // [{ name, manufacturer, internal_mm, external_mm, type? }]

/* UI state */
const selectedSADBrandKey = ref(null); // "<canonName>|<canonManu>"
const selectedSADSizeNum  = ref(null); // numeric size (e.g. 4)
const MIN_TOLERANCE_MM = 0.5;
const DEFAULT_TOLERANCE_MM = 1;
const tolerance = ref(DEFAULT_TOLERANCE_MM);
const selectedETTNameKeys = ref([]);
const loadError = ref(null);

/* ---- Normalisation & parsing (dedupe brand names) ----
   Past files included things like "AuraGain, ... device" and ® symbols, which must be normalised,
   otherwise duplicates appear even if they look "the same" to us.  :contentReference[oaicite:2]{index=2} */
function canonNameForKey(str) {
  return String(str || "")
    .replace(/[®™]/g, "")                                // remove marks
    .replace(/,?\s*Supraglottic Airway( Device)?/ig, "") // drop trailing descriptor
    .replace(/,?\s*Intubating Laryngeal Airway/ig, "")
    .replace(/\s+device$/i, "")                          // fix accidental trailing "device"
    .split(",")[0]                                       // keep model head
    .replace(/\s+/g, " ")
    .trim()
    .toLowerCase();
}
function canonManuForKey(str) {
  return String(str || "").replace(/\s+/g, " ").trim().toLowerCase();
}
/* For display we keep nice casing */
function displayName(str) {
  return (String(str || "").replace(/[®™]/g, "").split(",")[0] || "Unknown").trim();
}
function displayManu(str) {
  return String(str || "").trim();
}
function canonEttName(str) {
  return String(str || "")
    .replace(/[®™]/g, "")
    .replace(/\s+/g, " ")
    .trim()
    .toLowerCase();
}
/* Parse "Size 4, Adult…" => 4 */
function parseSizeNumber(val) {
  if (val == null) return null;
  if (typeof val === "number") return Number.isFinite(val) ? val : null;
  const m = String(val).match(/size\s*([0-9]+(?:\.[0-9]+)?)/i);
  return m ? Number(m[1]) : null;
}

/* Load */
onMounted(async () => {
  try {
    const baseUrl = new URL(import.meta.env.BASE_URL || "/", window.location.origin);
    const [sadsRes, ettsRes] = await Promise.all([
      fetch(new URL("sads.json", baseUrl)),
      fetch(new URL("etts.json", baseUrl)),
    ]);
    if (!sadsRes.ok || !ettsRes.ok) {
      throw new Error(`HTTP ${sadsRes.status}/${ettsRes.status}`);
    }
    sads.value = await sadsRes.json();
    etts.value = await ettsRes.json();
    loadError.value = null;
  } catch (err) {
    console.error("Failed to load SAD/ETT data", err);
    loadError.value = "Unable to load device data. Please refresh or try again later.";
    sads.value = [];
    etts.value = [];
  }
});

/* Brand list (strict dedupe) */
const sadBrands = computed(() => {
  const map = new Map();
  for (const row of sads.value) {
    const keyName = canonNameForKey(row.name);
    const keyManu = canonManuForKey(row.manufacturer);
    if (!keyName) continue;
    const key = `${keyName}|${keyManu}`;
    if (!map.has(key)) {
      map.set(key, {
        key,
        name: displayName(row.name),
        manufacturer: displayManu(row.manufacturer),
      });
    }
  }
  return Array.from(map.values()).sort(
    (a,b) => a.name.localeCompare(b.name) || a.manufacturer.localeCompare(b.manufacturer)
  );
});

/* Entries for selected brand */
const brandEntries = computed(() => {
  if (!selectedSADBrandKey.value) return [];
  const [wantName, wantManu] = selectedSADBrandKey.value.split("|");
  return sads.value.filter(
    r => `${canonNameForKey(r.name)}|${canonManuForKey(r.manufacturer)}` === `${wantName}|${wantManu}`
  );
});

/* Numeric size options: [3,4,5,…] */
const sadSizeOptions = computed(() => {
  const set = new Set();
  for (const r of brandEntries.value) {
    const n = parseSizeNumber(r.size);
    if (n != null) set.add(n);
  }
  return Array.from(set).sort((a,b)=>a-b);
});

const ettNameOptions = computed(() => {
  const map = new Map();
  for (const e of etts.value) {
    const key = canonEttName(e.name);
    if (!key) continue;
    if (!map.has(key)) {
      map.set(key, {
        key,
        label: displayName(e.name),
        manufacturer: displayManu(e.manufacturer),
      });
    }
  }
  return Array.from(map.values()).sort((a,b) => a.label.localeCompare(b.label));
});

/* Auto-pick largest size when brand changes (typical adult demo) */
watch([selectedSADBrandKey, sadSizeOptions], () => {
  selectedSADSizeNum.value = sadSizeOptions.value.length ? sadSizeOptions.value.at(-1) : null;
});

/* Clamp tolerance to minimum clearance */
watch(tolerance, (val) => {
  if (val < MIN_TOLERANCE_MM) tolerance.value = MIN_TOLERANCE_MM;
});

function showAllEttModels(checked) {
  if (checked) selectedETTNameKeys.value = [];
}

function scrollToStep(id) {
  const el = document.getElementById(id);
  if (el) el.scrollIntoView({ behavior: "smooth", block: "start" });
}
/* Selected entry (brand + size) -> gives us ID for maths */
const selectedSADEntry = computed(() => {
  if (!brandEntries.value.length || selectedSADSizeNum.value == null) return null;
  return brandEntries.value.find(r => parseSizeNumber(r.size) === Number(selectedSADSizeNum.value)) || null;
});

const sadID = computed(() => {
  if (!selectedSADEntry.value) return null;
  const v = Number(selectedSADEntry.value.internal_mm);
  return Number.isNaN(v) ? null : v;
});

/* ---- Compatibility table: largest & 2nd largest ETT per type ---- */
function getType(e){ return (e.type && String(e.type).trim()) || "ETT"; }

const tableRows = computed(() => {
  if (sadID.value == null) return [];

  const activeSet = selectedETTNameKeys.value.length
    ? new Set(selectedETTNameKeys.value)
    : null;

  const byName = new Map();
  for (const e of etts.value) {
    const key = canonEttName(e.name);
    if (!key) continue;
    if (activeSet && !activeSet.has(key)) continue;

    const id = Number(e.internal_mm);
    const od = Number(e.external_mm);
    if (Number.isNaN(id) || Number.isNaN(od)) continue;

    const gap = sadID.value - od;
    // Skip models that do not meet the current clearance requirement.
    if (gap < tolerance.value) continue;

    const entry = {
      id,
      od,
      gap,
      type: getType(e),
      model: displayName(e.name),
      manufacturer: displayManu(e.manufacturer || ""),
    };
    if (!byName.has(key)) byName.set(key, []);
    byName.get(key).push(entry);
  }

  const rows = [];
  for (const [key, list] of byName.entries()) {
    if (!list.length) continue;
    list.sort((a,b) => b.id - a.id || a.od - b.od);
    const topTwo = list.slice(0,2);
    for (const item of topTwo) {
      const isComfortable = item.gap >= tolerance.value + 0.25; // slightly above tolerance gets a green badge
      rows.push({
        nameKey: key,
        id: item.id,
        type: item.type,
        od: item.od,
        model: item.model,
        manufacturer: item.manufacturer,
        state: isComfortable ? "fit" : "tight"
      });
    }
  }

  return rows.sort(
    (a,b) => a.model.localeCompare(b.model) || b.id - a.id || a.od - b.od
  );
});
</script>

<template>
  <main class="container">
    <h1>Plan B Airway Compatibility Tool</h1>

    <p class="disclaimer">
      This app is intended to support estimation and should not replace clinical judgement.
      Data are taken from published sources and manufacturer information.
      External ETT diameter measurements are provided with the cuff in a deflated state.
      Always review device specifications, confirm compatibility, and check the actual fit before clinical use.
    </p>

    <div class="steps" aria-label="Steps to use the tool">
      <button class="step-card" type="button" @click="scrollToStep('step-brand')">
        <div class="step-badge">1</div>
        <div>
          <div class="step-title">Select SAD Model / Brand</div>
          <div class="step-sub">Pick the device to load available sizes</div>
        </div>
      </button>
      <button class="step-card" type="button" @click="scrollToStep('step-size')">
        <div class="step-badge">2</div>
        <div>
          <div class="step-title">Select SAD Size</div>
          <div class="step-sub">Choose the size you plan to use</div>
        </div>
      </button>
      <button class="step-card" type="button" @click="scrollToStep('step-filter')">
        <div class="step-badge">3</div>
        <div>
          <div class="step-title">Filter ETT Models</div>
          <div class="step-sub">Show all or narrow to specific tubes</div>
        </div>
      </button>
    </div>

    <p v-if="loadError" class="error">{{ loadError }}</p>

    <!-- 1) SAD model/brand -->
    <div class="field" id="step-brand">
      <label>1.Select SAD Model / Brand</label>
      <select v-model="selectedSADBrandKey">
        <option :value="null">— select model/brand —</option>
        <option v-for="b in sadBrands" :key="b.key" :value="b.key">
          {{ b.name }}<span v-if="b.manufacturer">, {{ b.manufacturer }}</span>
        </option>
      </select>
    </div>

    <!-- 2) SAD size (numbers only) -->
    <div class="field" id="step-size">
      <label>2.Select SAD Size</label>
      <select v-model="selectedSADSizeNum" :disabled="!selectedSADBrandKey">
        <option :value="null">— select size —</option>
        <option v-for="sz in sadSizeOptions" :key="sz" :value="sz">Size {{ sz }}</option>
      </select>

      <!-- Calculation box -->
      <div v-if="selectedSADEntry" class="calc">
        <strong>
          {{ displayName(selectedSADEntry.name) }}, Size {{ selectedSADSizeNum }}<br> 
           Internal Diameter {{ sadID?.toFixed(2) }} mm
        </strong>
      </div>
    </div>

    <!-- Tolerance slider -->
    <div class="field" v-if="selectedSADEntry">
      <label>Tolerance / clearance (mm): {{ tolerance.toFixed(2) }}</label>
      <input type="range" :min="MIN_TOLERANCE_MM" max="2" step="0.1" v-model.number="tolerance" />
      <small>Results must meet the tolerance. Green = comfortable (≥ tolerance + 0.25 mm); Yellow = just meets tolerance.</small>
    </div>

    <!-- ETT filter -->
    <div class="field" v-if="selectedSADEntry" id="step-filter">
      <label>3.Filter ETT models</label>
      <div class="tick-list">
        <label class="tick">
          <input
            type="checkbox"
            :checked="selectedETTNameKeys.length === 0"
            @change="showAllEttModels($event.target.checked)"
          >
          Show all
        </label>
        <label
          v-for="opt in ettNameOptions"
          :key="opt.key"
          class="tick"
        >
          <input
            type="checkbox"
            :value="opt.key"
            v-model="selectedETTNameKeys"
          >
          {{ opt.label }}
        </label>
      </div>
      <small>Tick models to narrow the list; leave blank for all models.</small>
    </div>

    <!-- Results -->
    <table v-if="selectedSADEntry" class="results">
      <caption>Largest compatible ETT sizes per selected model (top 2)</caption>
      <thead>
        <tr>
          <th>ETT model</th>
          <th>ETT size (ID mm)</th>
          <th>Type</th>
          <th>External Diameter (OD mm)</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="row in tableRows" :key="row.nameKey + row.id + row.od + row.type" :class="row.state">
          <td>{{ row.model }}</td>
          <td>{{ Number(row.id).toFixed(1) }}</td>
          <td>{{ row.type }}</td>
          <td>{{ row.od.toFixed(2) }}</td>
        </tr>
      </tbody>
    </table>

    <p v-if="selectedSADEntry && tableRows.length === 0" class="no-results">
      No compatible ETTs found for this SAD at the current tolerance.
    </p>
  </main>
</template>

<style>
body {
  background: #fff !important;
  color: #111;
  color-scheme: light;
}
:root { font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif; }
.container { max-width: 950px; margin: 2rem auto; padding: 1rem; }
h1 { font-size: 1.6rem; margin-bottom: 1rem; }

.disclaimer {
  margin: 0.5rem 0 1.25rem;
  font-size: 1.2rem;
  line-height: 1.4;
  color: #b71c1c;
  background: #fdecea;
  border-left: 4px solid #e74c3c;
  padding: 0.6rem 0.9rem;
  border-radius: 4px;
}

.steps {
  display: grid;
  gap: 12px;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  margin: 0.6rem 0 1.25rem;
}
.step-card {
  border: 1px solid #d3c7b8;
  background: #f6f1e8;
  border-radius: 12px;
  padding: 12px 14px 14px 14px;
  display: grid;
  grid-template-columns: auto 1fr;
  gap: 10px;
  align-items: center;
  cursor: pointer;
  transition: border-color 0.2s, transform 0.2s, box-shadow 0.2s;
  text-align: left;
}
.step-card:hover {
  border-color: #0c6d65;
  box-shadow: 0 4px 12px rgba(0,0,0,0.08);
  transform: translateY(-2px);
}
.step-card:focus-visible {
  outline: 2px solid #0c6d65;
  outline-offset: 2px;
}
.step-badge {
  width: 34px;
  height: 34px;
  border-radius: 50%;
  display: grid;
  place-items: center;
  background: #0c6d65;
  color: #fff;
  font-weight: 700;
}
.step-title { font-weight: 700; }
.step-sub { color: #3c3a36; font-size: 13px; }

.error {
  margin: 0.75rem 0;
  color: #8b0000;
  background: #fdecea;
  border-left: 4px solid #c62828;
  padding: 0.6rem 0.9rem;
  border-radius: 4px;
}

.field { margin-bottom: 1rem; }
.field label { display: block; font-weight: 700; margin-bottom: 0.4rem; }
select {
  width: 100%;
  padding: 0.45rem;
  font-size: 1.1rem;
}
select:disabled { background: #f5f5f5; color: #888; cursor: not-allowed; }
small { color: #666; }

.calc {
  margin-top: 0.6rem;
  background: #f9f9f9;
  padding: 0.5rem 0.75rem;
  border-radius: 6px;
  border: 1px solid #e3e3e3;
}

.tick-list {
  display: flex;
  flex-wrap: wrap;
  gap: 0.4rem 0.8rem;
  padding: 0.5rem;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  max-height: 230px;
  overflow-y: auto;
  background: #fafafa;
}
.tick {
  display: flex;
  align-items: center;
  gap: 0.35rem;
  font-size: 0.9rem;
}

.results {
  width: 100%;
  border-collapse: collapse;
  margin-top: 0.75rem;
}
.results caption {
  text-align: left; font-weight: 600; margin-bottom: 0.35rem; color: #333;
}
.results th, .results td {
  border: 1px solid #d9d9d9; padding: 0.55rem 0.75rem; text-align: left;
}
.results th { background: #f5f5f5; }
.results tr.fit   { background: #e9f7ef; }  /* green */
.results tr.tight { background: #fff8e6; }  /* yellow */

.no-results { margin-top: 1rem; font-style: italic; color: #666; }
</style>
