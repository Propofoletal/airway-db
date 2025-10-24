<script setup>
import { ref, computed, onMounted } from "vue";

/* Data (loaded from /public) */
const sads = ref([]);   // [{ name, manufacturer, size, internal_mm, notes }]
const etts = ref([]);   // [{ name, manufacturer, internal_mm, external_mm, type?, notes }]

/* Two-step SAD selection */
const selectedSADBrandKey = ref(null); // "<name>|<manufacturer>"
const selectedSADSize     = ref(null); // numeric size ONLY (e.g. 3, 4, 5)

/* Tolerance (mm) */
const tolerance = ref(0.5);

/* Load data */
onMounted(async () => {
  const [sadsRes, ettsRes] = await Promise.all([
    fetch("/sads.json"),
    fetch("/etts.json"),
  ]);
  sads.value = await sadsRes.json();
  etts.value = await ettsRes.json();
});

/* ---------- SAD model → sizes (numbers only) ---------- */

/** Distinct SAD model/brand options */
const sadBrands = computed(() => {
  const map = new Map();
  for (const s of sads.value) {
    const key = `${(s.name || "").trim()}|${(s.manufacturer || "").trim()}`;
    if (!map.has(key)) {
      map.set(key, { key, name: s.name || "Unknown", manufacturer: s.manufacturer || "" });
    }
  }
  return Array.from(map.values()).sort((a,b) =>
    (a.name||"").localeCompare(b.name||"") ||
    (a.manufacturer||"").localeCompare(b.manufacturer||"")
  );
});

/** Available size NUMBERS for the selected brand/model (unique, sorted) */
const sadSizeOptions = computed(() => {
  if (!selectedSADBrandKey.value) return [];
  const [n, m] = selectedSADBrandKey.value.split("|");
  const sizes = new Set();
  for (const s of sads.value) {
    if ((s.name||"").trim()===n && (s.manufacturer||"").trim()===(m||"")) {
      const sz = Number(s.size);
      if (!Number.isNaN(sz)) sizes.add(sz);
    }
  }
  return Array.from(sizes).sort((a,b)=>a-b);
});

/** The actual SAD entry for the selected brand + size (used to read ID etc.) */
const selectedSADEntry = computed(() => {
  if (!selectedSADBrandKey.value || selectedSADSize.value == null) return null;
  const [n, m] = selectedSADBrandKey.value.split("|");
  // pick the entry for that exact size; if multiple rows exist, take the first
  return sads.value.find(s =>
    (s.name||"").trim()===n &&
    (s.manufacturer||"").trim()===(m||"") &&
    Number(s.size) === Number(selectedSADSize.value)
  ) || null;
});

/** Convenience values */
const sadID = computed(() => {
  if (!selectedSADEntry.value) return null;
  const v = Number(selectedSADEntry.value.internal_mm);
  return Number.isNaN(v) ? null : v;
});

/* ---------- Table: largest & 2nd largest per ETT type ---------- */

function getType(e){ return (e.type && String(e.type).trim()) || "ETT"; }

/** Build rows:
 * - For each TYPE:
 *   - consider only entries that physically fit (gap > 0)
 *   - group by ETT ID (size) and choose the model with smallest OD at that size
 *   - take the two largest sizes (by ID)
 * - Colour based on tolerance (green/yellow). Non-fitting are hidden.
 */
const tableRows = computed(() => {
  if (sadID.value == null) return [];

  // collect candidates by type
  const byType = new Map();
  for (const e of etts.value) {
    const id = Number(e.internal_mm);
    const od = Number(e.external_mm);
    if (Number.isNaN(id) || Number.isNaN(od)) continue;

    const gap = sadID.value - od;
    if (gap <= 0) continue; // not physically passable

    const type = getType(e);
    if (!byType.has(type)) byType.set(type, []);
    byType.get(type).push({
      id, od, gap,
      name: e.name,
      manufacturer: e.manufacturer || ""
    });
  }

  const rows = [];

  for (const [type, entries] of byType.entries()) {
    if (!entries.length) continue;

    // group by ETT ID (size)
    const bySize = new Map();
    for (const ent of entries) {
      if (!bySize.has(ent.id)) bySize.set(ent.id, []);
      bySize.get(ent.id).push(ent);
    }

    // choose best model per size (min OD -> max gap)
    const sizeBest = [];
    for (const [id, list] of bySize.entries()) {
      const best = list.slice().sort((a,b) => a.od - b.od)[0];
      sizeBest.push(best);
    }

    // pick largest and second largest sizes
    sizeBest.sort((a,b) => b.id - a.id || a.od - b.od);
    const topTwo = sizeBest.slice(0, 2);

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

  // order: by type, then ID desc, then OD asc
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

    <!-- 1) Select SAD model/brand -->
    <div class="field">
      <label>Select SAD Model / Brand</label>
      <select
        v-model="selectedSADBrandKey"
        @change="selectedSADSize = null"
      >
        <option :value="null">— select model/brand —</option>
        <option v-for="b in sadBrands" :key="b.key" :value="b.key">
          {{ b.name }}<span v-if="b.manufacturer"> — {{ b.manufacturer }}</span>
        </option>
      </select>
    </div>

    <!-- 2) Select SAD size (numbers only; disabled until model picked) -->
    <div class="field">
      <label>Select SAD Size</label>
      <select v-model="selectedSADSize" :disabled="!selectedSADBrandKey">
        <option :value="null">— select size —</option>
        <option
          v-for="sz in sadSizeOptions"
          :key="sz"
          :value="sz"
        >
          Size {{ sz }}
        </option>
      </select>

      <!-- Calculation box (shows ID AFTER size selection) -->
      <div v-if="selectedSADEntry" class="calc">
        <strong>
          {{ selectedSADEntry.name }}
          <span v-if="selectedSADEntry.manufacturer">— {{ selectedSADEntry.manufacturer }}</span>
          — size {{ selectedSADSize }} — ID {{ sadID?.toFixed(2) }} mm
        </strong>
      </div>
    </div>

    <!-- Tolerance slider (below selectors) -->
    <div class="field" v-if="selectedSADEntry">
      <label>Tolerance / clearance (mm): {{ tolerance.toFixed(2) }}</label>
      <input type="range" min="0" max="2" step="0.1" v-model.number="tolerance" />
      <small>Green ≥ tolerance; Yellow &lt; tolerance. Non-fitting ETTs are hidden.</small>
    </div>

    <!-- Results table: largest & 2nd largest per type -->
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
