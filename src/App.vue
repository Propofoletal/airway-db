<script setup>
import { ref, computed, onMounted, watch } from "vue";

/* Data */
const sads = ref([]);   // [{ name, manufacturer, size:"Size 4, ...", internal_mm }]
const etts = ref([]);   // [{ name, manufacturer, internal_mm, external_mm, type? }]

/* UI state */
const selectedSADBrandKey = ref(null); // "<canonName>|<canonManu>"
const selectedSADSizeNum  = ref(null); // numeric size (e.g. 4)
const tolerance = ref(0.5);

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
/* Parse "Size 4, Adult…" => 4 */
function parseSizeNumber(val) {
  if (val == null) return null;
  if (typeof val === "number") return Number.isFinite(val) ? val : null;
  const m = String(val).match(/size\s*([0-9]+(?:\.[0-9]+)?)/i);
  return m ? Number(m[1]) : null;
}

/* Load */
onMounted(async () => {
  const [sadsRes, ettsRes] = await Promise.all([
    fetch("/sads.json"),
    fetch("/etts.json"),
  ]);
  sads.value = await sadsRes.json();
  etts.value = await ettsRes.json();
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

/* Auto-pick largest size when brand changes (typical adult demo) */
watch([selectedSADBrandKey, sadSizeOptions], () => {
  selectedSADSizeNum.value = sadSizeOptions.value.length ? sadSizeOptions.value.at(-1) : null;
});

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

  // collect candidates by type
  const byType = new Map();
  for (const e of etts.value) {
    const id = Number(e.internal_mm);
    const od = Number(e.external_mm);
    if (Number.isNaN(id) || Number.isNaN(od)) continue;

    const gap = sadID.value - od;
    if (gap <= 0) continue; // not passable

    const t = getType(e);
    if (!byType.has(t)) byType.set(t, []);
    byType.get(t).push({ id, od, gap, name: e.name, manufacturer: e.manufacturer || "" });
  }

  const rows = [];
  for (const [t, list] of byType.entries()) {
    if (!list.length) continue;

    // group by ETT ID (size) and pick best model (min OD) for that size
    const bySize = new Map();
    for (const x of list) {
      if (!bySize.has(x.id)) bySize.set(x.id, []);
      bySize.get(x.id).push(x);
    }
    const sizeBest = [];
    for (const [id, items] of bySize.entries()) {
      const best = items.slice().sort((a,b) => a.od - b.od)[0];
      sizeBest.push(best);
    }

    // largest + second largest sizes
    sizeBest.sort((a,b) => b.id - a.id || a.od - b.od);
    const topTwo = sizeBest.slice(0,2);

    for (const r of topTwo) {
      rows.push({
        id: r.id,
        type: t,
        od: r.od,
        model: r.name,
        manufacturer: r.manufacturer,
        state: r.gap >= tolerance.value ? "fit" : "tight"
      });
    }
  }

  // order: type, then size desc, then OD asc
  return rows.sort(
    (a,b) => (a.type||"").localeCompare(b.type||"") || b.id - a.id || a.od - b.od
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
          {{ b.name }}<span v-if="b.manufacturer">, {{ b.manufacturer }}</span>
        </option>
      </select>
    </div>

    <!-- 2) SAD size (numbers only) -->
    <div class="field">
      <label>Select SAD Size</label>
      <select v-model="selectedSADSizeNum" :disabled="!selectedSADBrandKey">
        <option :value="null">— select size —</option>
        <option v-for="sz in sadSizeOptions" :key="sz" :value="sz">Size {{ sz }}</option>
      </select>

      <!-- Calculation box -->
      <div v-if="selectedSADEntry" class="calc">
        <strong>
          {{ displayName(selectedSADEntry.name) }}
          // <span v-if="selectedSADEntry.manufacturer"></span>
          </br> Size {{ selectedSADSizeNum }} </br> Internal Diameter {{ sadID?.toFixed(2) }} mm
        </strong>
      </div>
    </div>

    <!-- Tolerance slider -->
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
