<script setup>
import { ref, computed, onMounted, watch } from "vue";

/* Data */
const sads = ref([]);   // from /public/sads.json (size is a string like "Size 4, ...")
const etts = ref([]);   // from /public/etts.json  (expects {name, manufacturer, internal_mm, external_mm, type?})

/* UI state */
const selectedSADBrandKey = ref(null); // "<canonName>|<canonManu>"
const selectedSADSizeNum  = ref(null); // numeric size (e.g. 4) parsed from "Size 4, ..."
const tolerance = ref(0.5);

/* ---- utils to normalize data from your sads.json ---- */
function parseSizeNumber(s) {
  // "Size 4, Medium adult, 50-90 kg" -> 4
  if (s == null) return null;
  const m = String(s).match(/size\s*([0-9]+(?:\.[0-9]+)?)/i);
  return m ? Number(m[1]) : null;
}
function canon(str) {
  return String(str || "")
    .replace(/[®™]/g, "")   // remove marks
    .trim();
}
function canonManu(manu) {
  // fix common misspelling "Intersurgcial" -> "Intersurgical"
  const c = canon(manu);
  if (/^intersurg/i.test(c) && c !== "Intersurgical") return "Intersurgical";
  return c;
}

/* Load JSONs */
onMounted(async () => {
  const [sadsRes, ettsRes] = await Promise.all([
    fetch("/sads.json"),
    fetch("/etts.json"),
  ]);
  sads.value = await sadsRes.json();
  etts.value = await ettsRes.json();
});

/* ---- Build brand list from sads.json (tolerant) ---- */
const sadBrands = computed(() => {
  const map = new Map();
  for (const row of sads.value) {
    // Brand key = full canonical model name + canonical manufacturer
    const name = canon(row.name);
    const manu = canonManu(row.manufacturer);
    const key  = `${name}|${manu}`;
    if (!map.has(key)) {
      // display name: shorten very long model labels to text before first comma
      const shortName = name.split(",")[0].trim();
      map.set(key, { key, name: shortName || name, manufacturer: manu });
    }
  }
  return Array.from(map.values()).sort((a,b) =>
    a.name.localeCompare(b.name) || a.manufacturer.localeCompare(b.manufacturer)
  );
});

/* ---- Sizes available for selected brand (parsed numbers only) ---- */
const brandEntries = computed(() => {
  if (!selectedSADBrandKey.value) return [];
  const [wantName, wantManu] = selectedSADBrandKey.value.split("|");
  return sads.value.filter(r => canon(r.name) === wantName && canonManu(r.manufacturer) === wantManu);
});

const sadSizeOptions = computed(() => {
  const set = new Set();
  for (const r of brandEntries.value) {
    const n = parseSizeNumber(r.size);
    if (n != null) set.add(n);
  }
  return Array.from(set).sort((a,b)=>a-b);
});

/* Auto-pick a size when brand changes (e.g. largest adult size) */
watch([selectedSADBrandKey, sadSizeOptions], () => {
  selectedSADSizeNum.value = sadSizeOptions.value.length ? sadSizeOptions.value.at(-1) : null;
});

/* The exact SAD row for brand + size (used for ID) */
const selectedSADEntry = computed(() => {
  if (!brandEntries.value.length || selectedSADSizeNum.value == null) return null;
  return brandEntries.value.find(r => parseSizeNumber(r.size) === Number(selectedSADSizeNum.value)) || null;
});

/* Convenience */
const sadID = computed(() => {
  if (!selectedSADEntry.value) return null;
  const v = Number(selectedSADEntry.value.internal_mm);
  return Number.isNaN(v) ? null : v;
});

/* -------- Table: largest & 2nd largest per ETT type (same behaviour) -------- */
function getType(e){ return (e.type && String(e.type).trim()) || "ETT"; }

const tableRows = computed(() => {
  if (sadID.value == null) return [];
  const byType = new Map();

  for (const e of etts.value) {
    const id = Number(e.internal_mm);
    const od = Number(e.external_mm);
    if (Number.isNaN(id) || Number.isNaN(od)) continue;
    const gap = sadID.value - od;
    if (gap <= 0) continue; // not passable

    const type = getType(e);
    if (!byType.has(type)) byType.set(type, []);
    byType.get(type).push({ id, od, gap, name: e.name, manufacturer: e.manufacturer || "" });
  }

  const rows = [];
  for (const [type, entries] of byType.entries()) {
    if (!entries.length) continue;

    // group by ETT ID and pick best model (min OD) for that ID
    const bySize = new Map();
    for (const ent of entries) {
      if (!bySize.has(ent.id)) bySize.set(ent.id, []);
      bySize.get(ent.id).push(ent);
    }
    const sizeBest = [];
    for (const [id, list] of bySize.entries()) {
      const best = list.slice().sort((a,b)=>a.od-b.od)[0];
      sizeBest.push(best);
    }

    // take largest and second-largest sizes
    sizeBest.sort((a,b)=> b.id - a.id || a.od - b.od);
    const topTwo = sizeBest.slice(0,2);

    for (const r of topTwo) {
      rows.push({
        id: r.id,
        type,
        od: r.od,
        model: r.name,
        manufacturer: r.manufacturer,
        state: r.gap >= tolerance.value ? "fit" : "tight"
      });
    }
  }

  return rows.sort((a,b) =>
    (a.type||"").localeCompare(b.type||"") ||
    b.id - a.id ||
    a.od - b.od
  );
});
</script>

<template>
  <main class="container">
    <h1>Airway Compatibility — SAD → ETT</h1>

    <p class="disclaimer">
      ⚠️ This app is intended for estimation only.
      Data are based on information provided by manufacturers.
      Always check device specifications and confirm compatibility.
      Always check actual device fit before clinical use.
    </p>

    <!-- 1) SAD model/brand -->
    <div class="field">
      <label>Select SAD Model / Brand</label>
      <select v-model="selectedSADBrandKey">
        <option :value="null">— select model/brand —</option>
        <option v-for="b in sadBrands" :key="b.key" :value="b.key">
          {{ b.name }}<span v-if="b.manufacturer"> — {{ b.manufacturer }}</span>
        </option>
      </select>
    </div>

    <!-- 2) SAD size (numeric, parsed from "Size X") -->
    <div class="field">
      <label>Select SAD Size</label>
      <select v-model="selectedSADSizeNum" :disabled="!selectedSADBrandKey">
        <option :value="null">— select size —</option>
        <option v-for="sz in sadSizeOptions" :key="sz" :value="sz">
          Size {{ sz }}
        </option>
      </select>

      <!-- Calc box -->
      <div v-if="selectedSADEntry" class="calc">
        <strong>
          {{ selectedSADEntry.name.split(',')[0] }}
          <span v-if="selectedSADEntry.manufacturer">— {{ selectedSADEntry.manufacturer }}</span>
          — size {{ selectedSADSizeNum }} — ID {{ sadID?.toFixed(2) }} mm
        </strong>
      </div>
    </div>

    <!-- Tolerance -->
    <div class="field" v-if="selectedSADEntry">
      <label>Tolerance / clearance (mm): {{ tolerance.toFixed(2) }}</label>
      <input type="range" min="0" max="2" step="0.1" v-model.number="tolerance" />
      <small>Green ≥ tolerance; Yellow &lt; tolerance. Non-fitting ETTs are hidden.</small>
    </div>

    <!-- Results -->
    <table v-if="selectedSADEntry" class="results">
      <caption>Largest compatible ETT sizes per type (top 2)</caption>
      <thead>
        <tr>
          <th>ETT size (ID mm)</th>
          <th>Type</th>
          <th>External Diameter (OD mm)</th>
          <th>Model / Manufacturer</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="row in tableRows" :key="row.type + row.model + row.id + row.od" :class="row.state">
          <td>{{ Number(row.id).toFixed(1) }}</td>
          <td>{{ row.type }}</td>
          <td>{{ row.od.toFixed(2) }}</td>
          <td>
            {{ row.model }}
            <span v-if="row.manufacturer"> — {{ row.manufacturer }}</span>
          </td>
        </tr>
      </tbody>
    </table>

    <p v-if="selectedSADEntry && tableRows.length === 0" class="no-results">
      No compatible ETTs found for this SAD at the current tolerance.
    </p>
  </main>
</template>

<style>
:root { font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif; }
.container { max-width: 950px; margin: 2rem auto; padding: 1rem; }
h1 { font-size: 1.6rem; margin-bottom: 1rem; }

.disclaimer {
  margin: 0.5rem 0 1.25rem;
  font-size: 0.9rem;
  line-height: 1.4;
  color: #b71c1c;
  background: #fdecea;
  border-left: 4px solid #e74c3c;
  padding: 0.6rem 0.9rem;
  border-radius: 4px;
}

.field { margin-bottom: 1rem; }
.field label { display: block; font-weight: 600; margin-bottom: 0.4rem; }
select { width: 100%; padding: 0.45rem; }
select:disabled { background: #f5f5f5; color: #888; cursor: not-allowed; }
small { color: #666; }

.calc {
  margin-top: 0.6rem;
  background: #f9f9f9;
  padding: 0.5rem 0.75rem;
  border-radius: 6px;
  border: 1px solid #e3e3e3;
}

.results { width: 100%; border-collapse: collapse; margin-top: 0.75rem; }
.results caption { text-align: left; font-weight: 600; margin-bottom: 0.35rem; color: #333; }
.results th, .results td { border: 1px solid #d9d9d9; padding: 0.55rem 0.75rem; text-align: left; }
.results th { background: #f5f5f5; }

.results tr.fit   { background: #e9f7ef; }  /* green */
.results tr.tight { background: #fff8e6; }  /* yellow */

.no-results { margin-top: 1rem; font-style: italic; color: #666; }
</style>
