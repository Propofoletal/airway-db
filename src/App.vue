<script setup>
import { ref, computed, onMounted } from "vue";

const sads = ref([]);   // [{ name, manufacturer, internal_mm, notes }]
const etts = ref([]);   // [{ name, manufacturer, internal_mm, external_mm, notes, type? }]

/* SAD selection (two-step) */
const selectedSADBrandKey = ref(null); // brand/model key
const selectedSADSize = ref(null);     // a specific SAD entry (object with internal_mm)

/* Tolerance slider (mm) — placed BELOW selectors per your choice (B) */
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

/* ---- Helpers for SAD brand → sizes ---- */

/** Unique SAD brands/models (by name + manufacturer) */
const sadBrands = computed(() => {
  const map = new Map();
  for (const s of sads.value) {
    const key = `${(s.name || "").trim()}|${(s.manufacturer || "").trim()}`;
    if (!map.has(key)) {
      map.set(key, {
        key,
        name: s.name || "Unknown",
        manufacturer: s.manufacturer || "",
      });
    }
  }
  // sort alphabetically by name then manufacturer
  return Array.from(map.values()).sort((a, b) =>
    (a.name || "").localeCompare(b.name || "") ||
    (a.manufacturer || "").localeCompare(b.manufacturer || "")
  );
});

/** Sizes (entries) available for the selected brand/model */
const sadSizesForBrand = computed(() => {
  if (!selectedSADBrandKey.value) return [];
  const [brandName, brandManu] = selectedSADBrandKey.value.split("|");
  return sads.value
    .filter(
      s =>
        (s.name || "").trim() === brandName &&
        (s.manufacturer || "").trim() === (brandManu || "")
    )
    .sort((a, b) => Number(a.internal_mm) - Number(b.internal_mm));
});

/** Convenience: selected SAD ID (internal diameter) */
const selectedSAD_ID = computed(() => {
  if (!selectedSADSize.value) return null;
  const v = Number(selectedSADSize.value.internal_mm);
  return Number.isNaN(v) ? null : v;
});

/* ---- Build table of largest-fitting ETT per model ---- */

/** Group ETTs by model/manufacturer; return map key → array of sizes (entries) */
function groupETTsByModel(ettsArr) {
  const map = new Map();
  for (const e of ettsArr) {
    const key = `${(e.name || "").trim()}|${(e.manufacturer || "").trim()}`;
    if (!map.has(key)) map.set(key, []);
    map.get(key).push(e);
  }
  return map;
}

/** Rows for table: only the largest ID that fits per ETT model */
const largestFitRows = computed(() => {
  const sadID = selectedSAD_ID.value;
  if (sadID == null) return [];

  const modelGroups = groupETTsByModel(etts.value);
  const rows = [];

  for (const [key, entries] of modelGroups.entries()) {
    // For this model, find all entries that physically fit at all (gap > 0)
    const fitEntries = entries
      .map(e => {
        const id = Number(e.internal_mm);
        const od = Number(e.external_mm);
        if (Number.isNaN(id) || Number.isNaN(od)) return null;
        const gap = sadID - od;
        if (gap <= 0) return null; // won't fit
        return { id, od, gap, type: e.type || "ETT", name: e.name, manufacturer: e.manufacturer || "", notes: e.notes || "" };
      })
      .filter(Boolean);

    if (!fitEntries.length) continue;

    // Choose the largest ID that still fits
    fitEntries.sort((a, b) => b.id - a.id || a.od - b.od);
    const best = fitEntries[0];

    // Colour logic: green if gap ≥ tolerance; yellow if 0 < gap < tolerance
    const state = best.gap >= tolerance.value ? "fit" : "tight";

    rows.push({
      id: best.id,
      type: best.type,
      od: best.od,
      model: best.name,
      manufacturer: best.manufacturer,
      state,
    });
  }

  // Sort rows by ETT size (ID) desc, then OD asc, then model
  rows.sort((a, b) => b.id - a.id || a.od - b.od || (a.model || "").localeCompare(b.model || ""));
  return rows;
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

    <!-- Step 1: Select SAD brand/model -->
    <div class="field">
      <label>Select SAD Brand / Model</label>
      <select
        v-model="selectedSADBrandKey"
        @change="selectedSADSize = null"
      >
        <option :value="null">— select brand/model —</option>
        <option
          v-for="b in sadBrands"
          :key="b.key"
          :value="b.key"
        >
          {{ b.name }}<span v-if="b.manufacturer"> — {{ b.manufacturer }}</span>
        </option>
      </select>
    </div>

    <!-- Step 2: Select SAD size (filtered by brand) -->
    <div class="field" v-if="selectedSADBrandKey">
      <label>Select SAD Size</label>
      <select v-model="selectedSADSize">
        <option :value="null">— select size —</option>
        <option
          v-for="s in sadSizesForBrand"
          :key="s.name + s.internal_mm + (s.manufacturer||'')"
          :value="s"
        >
          ID {{ Number(s.internal_mm).toFixed(2) }} mm
        </option>
      </select>
      <p v-if="selectedSAD_ID !== null" class="sad-id">
        Selected SAD internal diameter: <strong>{{ selectedSAD_ID.toFixed(2) }} mm</strong>
      </p>
    </div>

    <!-- Tolerance slider (placed BELOW selectors) -->
    <div class="field" v-if="selectedSADSize">
      <label>Tolerance / clearance (mm): {{ tolerance.toFixed(2) }}</label>
      <input type="range" min="0" max="1.5" step="0.1" v-model.number="tolerance" />
      <small>Green ≥ tolerance. Yellow &lt; tolerance. Rows that don't fit are hidden.</small>
    </div>

    <!-- Results table -->
    <table v-if="selectedSADSize" class="results">
      <caption>Showing largest compatible ETT per model</caption>
      <thead>
        <tr>
          <th>ETT size (ID mm)</th>
          <th>Type</th>
          <th>External Diameter (OD mm)</th>
          <th>Model / Manufacturer</th>
        </tr>
      </thead>
      <tbody>
        <tr
          v-for="row in largestFitRows"
          :key="row.model + row.manufacturer + row.id"
          :class="row.state"
        >
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

    <p v-if="selectedSADSize && largestFitRows.length === 0" class="no-results">
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
small { color: #666; }
.sad-id { margin-top: 0.35rem; color: #333; }

.results {
  width: 100%;
  border-collapse: collapse;
  margin-top: 0.75rem;
}
.results caption {
  text-align: left;
  font-weight: 600;
  margin-bottom: 0.35rem;
  color: #333;
}
.results th, .results td {
  border: 1px solid #d9d9d9;
  padding: 0.55rem 0.75rem;
  text-align: left;
}
.results th { background: #f5f5f5; }

.results tr.fit   { background: #e9f7ef; }  /* green */
.results tr.tight { background: #fff8e6; }  /* yellow */

.no-results { margin-top: 1rem; font-style: italic; color: #666; }
</style>
