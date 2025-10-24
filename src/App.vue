<script setup>
import { ref, computed, onMounted } from "vue";

const sads = ref([]);
const etts = ref([]);

const selectedSAD = ref(null);
const clearance = 0.5; // mm threshold for comfortable fit

onMounted(async () => {
  const [sadsRes, ettsRes] = await Promise.all([
    fetch("/sads.json"),
    fetch("/etts.json"),
  ]);
  sads.value = await sadsRes.json();
  etts.value = await ettsRes.json();
});

/* Compute compatible ETTs */
const compatibleETTs = computed(() => {
  if (!selectedSAD.value) return [];
  const sadID = Number(selectedSAD.value.internal_mm);
  if (Number.isNaN(sadID)) return [];

  return etts.value
    .map((e) => {
      const od = Number(e.external_mm);
      const gap = sadID - od;
      if (gap <= 0) return null;
      const state = gap >= clearance ? "fit" : "tight";
      return {
        id: e.internal_mm,
        type: e.type || "ETT",
        od,
        model: e.name,
        manufacturer: e.manufacturer || "",
        state,
      };
    })
    .filter(Boolean)
    .sort((a, b) => a.id - b.id || a.od - b.od);
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

    <div class="field">
      <label>Choose Supraglottic Airway Device (SAD)</label>
      <select v-model="selectedSAD">
        <option :value="null">— select a SAD —</option>
        <option v-for="s in sads" :key="s.name + s.internal_mm" :value="s">
          {{ s.name }}
          <span v-if="s.manufacturer">— {{ s.manufacturer }}</span>
          — ID {{ Number(s.internal_mm).toFixed(2) }} mm
        </option>
      </select>
    </div>

    <table v-if="selectedSAD" class="results">
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
          v-for="e in compatibleETTs"
          :key="e.model + e.od"
          :class="e.state"
        >
          <td>{{ e.id }}</td>
          <td>{{ e.type }}</td>
          <td>{{ e.od.toFixed(2) }}</td>
          <td>
            {{ e.model }}
            <span v-if="e.manufacturer">— {{ e.manufacturer }}</span>
          </td>
        </tr>
      </tbody>
    </table>

    <p v-if="selectedSAD && compatibleETTs.length === 0" class="no-results">
      No ETTs fit through this SAD (based on current data).
    </p>
  </main>
</template>

<style>
:root {
  font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif;
}
.container {
  max-width: 900px;
  margin: 2rem auto;
  padding: 1rem;
}
h1 {
  font-size: 1.6rem;
  margin-bottom: 1rem;
}
.field {
  margin-bottom: 1rem;
}
label {
  font-weight: 600;
  display: block;
  margin-bottom: 0.4rem;
}
select {
  width: 100%;
  padding: 0.4rem;
}
.disclaimer {
  margin: 0.5rem 0 1.5rem;
  font-size: 0.9rem;
  line-height: 1.4;
  color: #b71c1c;
  background: #fdecea;
  border-left: 4px solid #e74c3c;
  padding: 0.6rem 0.9rem;
  border-radius: 4px;
}
.results {
  width: 100%;
  border-collapse: collapse;
  margin-top: 1rem;
}
.results th,
.results td {
  border: 1px solid #ccc;
  padding: 0.5rem 0.75rem;
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
</style>
