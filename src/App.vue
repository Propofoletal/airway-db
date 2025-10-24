<script setup>
import { ref, computed, onMounted } from "vue";

const sads = ref([]);
const etts = ref([]);

const selectedSADBrandKey = ref(null);
const selectedSADEntry = ref(null);
const tolerance = ref(0.5);

onMounted(async () => {
  const [sadsRes, ettsRes] = await Promise.all([
    fetch("/sads.json"),
    fetch("/etts.json"),
  ]);
  sads.value = await sadsRes.json();
  etts.value = await ettsRes.json();
});

/* --- SAD selection logic --- */
const sadBrands = computed(() => {
  const map = new Map();
  for (const s of sads.value) {
    const key = `${(s.name || "").trim()}|${(s.manufacturer || "").trim()}`;
    if (!map.has(key)) map.set(key, { key, name: s.name, manufacturer: s.manufacturer || "" });
  }
  return Array.from(map.values()).sort((a, b) =>
    (a.name || "").localeCompare(b.name || "") ||
    (a.manufacturer || "").localeCompare(b.manufacturer || "")
  );
});

const sadSizeEntries = computed(() => {
  if (!selectedSADBrandKey.value) return [];
  const [n, m] = selectedSADBrandKey.value.split("|");
  return sads.value
    .filter(s => (s.name || "").trim() === n && (s.manufacturer || "").trim() === (m || ""))
    .sort((a, b) => Number(a.size || a.internal_mm) - Number(b.size || b.internal_mm));
});

const sadID = computed(() => {
  if (!selectedSADEntry.value) return null;
  const v = Number(selectedSADEntry.value.internal_mm);
  return Number.isNaN(v) ? null : v;
});

const sadSizeLabel = computed(() => {
  if (!selectedSADEntry.value) return "";
  const s = selectedSADEntry.value.size;
  return s ? `size ${s}` : `ID ${Number(selectedSADEntry.value.internal_mm).toFixed(2)} mm`;
});

/* --- Dynamic table --- */
function getType(e) {
  return (e.type && String(e.type).trim()) || "ETT";
}

const tableRows = computed(() => {
  if (sadID.value == null) return [];

  const byType = new Map();
  for (const e of etts.value) {
    const id = Number(e.internal_mm);
    const od = Number(e.external_mm);
    if (Number.isNaN(id) || Number.isNaN(od)) continue;

    const gap = sadID.value - od;
    if (gap <= 0) continue; // no fit

    const type = getType(e);
    if (!byType.has(type)) byType.set(type, []);
    byType.get(type).push({
      id,
      od,
      gap,
      name: e.name,
      manufacturer: e.manufacturer || "",
    });
  }

  const rows = [];

  for (const [type, entries] of byType.entries()) {
    if (!entries.length) continue;
    entries.sort((a, b) => b.id - a.id || a.od - b.od);

    // Dynamic rule: include all fits that meet current tolerance
    const fits = entries.filter(r => r.gap >= 0);
    const dynamicFits = fits.filter(r => r.gap >= 0);
    const shown = dynamicFits
      .filter(r => r.gap > 0)
      .slice(0, 5); // show up to 5 best matches per type

    for (const r of shown) {
      rows.push({
        id: r.id,
        type,
        od: r.od,
        model: r.name,
        manufacturer: r.manufacturer,
        gap: r.gap,
        state: r.gap >= tolerance.value ? "fit" : "tight",
      });
    }
  }

  return rows.sort(
    (a, b) =>
      (a.type || "").localeCompare(b.type || "") ||
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
    @change="selectedSADEntry = null"
  >
    <option :value="null">— select model/brand —</option>
    <option v-for="b in sadBrands" :key="b.key" :value="b.key">
      {{ b.name }}<span v-if="b.manufacturer"> — {{ b.manufacturer }}</span>
    </option>
  </select>
</div>

<!-- 2) Select SAD size (disabled until model chosen) -->
<div class="field">
  <label>Select SAD Size</label>
  <select
    v-model="selectedSADEntry"
    :disabled="!selectedSADBrandKey"
  >
    <option :value="null">— select size —</option>
    <option
      v-for="s in sadSizeEntries"
      :key="s.name + (s.size??'') + s.internal_mm + (s.manufacturer||'')"
      :value="s"
    >
      <template v-if="s.size !== undefined && s.size !== null && `${s.size}`.trim() !== ''">
        Size {{ s.size }} — ID {{ Number(s.internal_mm).toFixed(2) }} mm
      </template>
      <template v-else>
        ID {{ Number(s.internal_mm).toFixed(2) }} mm
      </template>
    </option>
  </select>

  <!-- Calculation box appears only after a size is picked -->
  <div v-if="selectedSADEntry" class="calc">
    <strong>
      {{ selectedSADEntry.name }}
      <span v-if="selectedSADEntry.manufacturer">— {{ selectedSADEntry.manufacturer }}</span>
      — {{ sadSizeLabel }} — ID {{ sadID?.toFixed(2) }} mm
    </strong>
  </div>
</div>

<!-- 3) Tolerance slider shows only after size is set -->
<div class="field" v-if="selectedSADEntry">
  <label>Tolerance / clearance (mm): {{ tolerance.toFixed(2) }}</label>
  <input type="range" min="0" max="2" step="0.1" v-model.number="tolerance" />
  <small>Adjust tolerance: increasing may reveal larger ETTs that now fit.</small>
</div>


    <!-- Dynamic table -->
    <transition-group name="fade" tag="table" v-if="selectedSADEntry" class="results">
      <caption>Compatible ETTs (auto-updates as tolerance changes)</caption>
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
          v-for="row in tableRows"
          :key="row.type + row.model + row.id + row.od"
          :class="row.state"
        >
          <td>{{ row.id.toFixed(1) }}</td>
          <td>{{ row.type }}</td>
          <td>{{ row.od.toFixed(2) }}</td>
          <td>
            {{ row.model }}
            <span v-if="row.manufacturer"> — {{ row.manufacturer }}</span>
          </td>
        </tr>
      </tbody>
    </transition-group>

    <p v-if="selectedSADEntry && tableRows.length === 0" class="no-results">
      No compatible ETTs found for this SAD at the current tolerance.
    </p>
  </main>
</template>

<style>
:root {
  font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif;
}
.container {
  max-width: 950px;
  margin: 2rem auto;
  padding: 1rem;
}
h1 {
  font-size: 1.6rem;
  margin-bottom: 1rem;
}
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
.field {
  margin-bottom: 1rem;
}
.field label {
  display: block;
  font-weight: 600;
  margin-bottom: 0.4rem;
}
select {
  width: 100%;
  padding: 0.45rem;
}
small {
  color: #666;
}
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
  text-align: left;
  font-weight: 600;
  margin-bottom: 0.35rem;
  color: #333;
}
.results th,
.results td {
  border: 1px solid #d9d9d9;
  padding: 0.55rem 0.75rem;
  text-align: left;
}
.results th {
  background: #f5f5f5;
}
.results tr.fit {
  background: #e9f7ef;
}
.results tr.tight {
  background: #fff8e6;
}
.no-results {
  margin-top: 1rem;
  font-style: italic;
  color: #666;
}

/* Smooth fade for table updates */
.fade-enter-active,
.fade-leave-active {
  transition: all 0.4s ease;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
  transform: translateY(10px);
}
</style>
