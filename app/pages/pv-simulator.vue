<script setup>
import { computed, reactive, ref, watch } from 'vue'
import UiParam from '@/components/UiParam.vue'
import UiParamSmall from '@/components/UiParamSmall.vue'
import UiKpi from '@/components/UiKpi.vue'
import UiEchart from '@/components/UiEchart.vue'

const MONTH_SHARES = [2.5, 3.5, 7.5, 10.5, 12.5, 12.5, 12.0, 11.0, 8.5, 6.5, 2.9, 2.6]
const DAYS_IN_MONTH = [31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31]

function toFixedNice(n, frac = 1) {
  return Number.isFinite(n) ? n.toFixed(frac) : '–'
}

const state = reactive({
  kwp: 50,
  specificYield: 900,
  dailyLoad: 30,
  batteryKwh: 300,
  usableDoD: 90,
  rtEff: 90,
  inverterEff: 97,
  addRandomness: true,
  randomSeed: 1,
})

const usableCap = computed(() => (state.batteryKwh * Math.max(0, Math.min(state.usableDoD, 100))) / 100)

function mulberry32(a) {
  return function () {
    let t = (a += 0x6d2b79f5)
    t = Math.imul(t ^ (t >>> 15), t | 1)
    t ^= t + Math.imul(t ^ (t >>> 7), t | 61)
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296
  }
}

function rollingWindowDeficit(points, window) {
  let worst = { window, unmet: 0, startDay: 1 }
  for (let i = 0; i <= points.length - window; i++) {
    const slice = points.slice(i, i + window)
    const unmet = slice.reduce((a, p) => a + p.unmet, 0)
    if (unmet > worst.unmet) worst = { window, unmet, startDay: i + 1 }
  }
  return worst
}

function simulate() {
  const rng = mulberry32(state.randomSeed)
  const annualPV = state.kwp * state.specificYield
  const totalShares = MONTH_SHARES.reduce((a, b) => a + b, 0)

  const points = []
  let dayOfYear = 1
  let soc = usableCap.value / 2
  const rt = Math.max(0, Math.min(state.rtEff, 100)) / 100
  const inv = Math.max(0, Math.min(state.inverterEff, 100)) / 100

  for (let m = 0; m < 12; m++) {
  const days = DAYS_IN_MONTH[m]
  const share = MONTH_SHARES[m]
    const monthEnergy = (annualPV * share) / totalShares
    const baseDaily = monthEnergy / days
    for (let d = 0; d < days; d++) {
      let variability = 1
      if (state.addRandomness) {
        const winterFactor = [0, 1, 10, 11].includes(m) ? 0.30 : 0.15
        const r = rng() * 2 - 1
        variability = 1 + r * winterFactor
        if (variability < 0) variability = 0
      }
      const pvAC = baseDaily * variability * inv
      const load = state.dailyLoad
      const direct = Math.min(pvAC, load)
      let surplus = Math.max(0, pvAC - direct)
      const deficit = Math.max(0, load - direct)

      const eta = Math.sqrt(rt)
      const chargePossible = Math.max(0, usableCap.value - soc)
      const chargeToStore = Math.min(chargePossible, surplus * eta)
      soc += chargeToStore
      surplus -= chargeToStore / eta

      const dischargePossible = soc * eta
      const discharge = Math.min(dischargePossible, deficit)
      const unmet = Math.max(0, deficit - discharge)
      soc -= discharge / eta

      points.push({
        day: dayOfYear,
        month: m + 1,
        pv: pvAC,
        load,
        net: pvAC - load,
        soc,
        charge: chargeToStore,
        discharge,
        unmet,
      })
      dayOfYear++
    }
  }

  const totalLoad = points.reduce((a, p) => a + p.load, 0)
  const unmetLoad = points.reduce((a, p) => a + p.unmet, 0)
  const autonomy = totalLoad > 0 ? 100 * (1 - unmetLoad / totalLoad) : 100
  const maxSOC = points.reduce((m, p) => Math.max(m, p.soc), 0)
  const minSOC = points.reduce((m, p) => Math.min(m, p.soc), usableCap.value)
  const darkest7 = rollingWindowDeficit(points, 7)
  const darkest14 = rollingWindowDeficit(points, 14)

  return { points, autonomy, totalLoad, unmetLoad, maxSOC, minSOC, darkest7, darkest14 }
}

const sim = computed(simulate)

// Force-remount charts on parameter changes to avoid heavy internal updates
const chartKey = ref(0)
watch(
  () => [
    state.kwp,
    state.specificYield,
    state.dailyLoad,
    state.batteryKwh,
    state.usableDoD,
    state.rtEff,
    state.inverterEff,
    state.addRandomness,
    state.randomSeed,
  ],
  () => {
    chartKey.value++
  },
  { flush: 'post' },
)

function resetDefaults() {
  state.kwp = 50
  state.specificYield = 900
  state.dailyLoad = 30
  state.batteryKwh = 300
  state.usableDoD = 90
  state.rtEff = 90
  state.inverterEff = 97
  state.addRandomness = true
  state.randomSeed = 1
}

const chartData = computed(() =>
  sim.value.points.map((p) => ({
    day: p.day,
    pv: Number(p.pv.toFixed(1)),
    load: Number(p.load.toFixed(1)),
    net: Number(p.net.toFixed(1)),
    soc: Number(p.soc.toFixed(1)),
    unmet: Number(p.unmet.toFixed(1)),
  })),
)

const yearPV = computed(() => state.kwp * state.specificYield)
// ApexCharts series/options
const socOption = computed(() => ({
  color: ['#6366f1'],
  animation: false,
  grid: { left: 45, right: 20, top: 20, bottom: 40 },
  textStyle: { color: '#334155' },
  xAxis: {
    type: 'category',
    data: chartData.value.map((d) => d.day),
    name: 'Tag',
    nameLocation: 'middle',
    nameGap: 30,
  },
  yAxis: {
    type: 'value',
    min: 0,
    max: Math.max(1, usableCap.value),
    name: 'kWh',
  },
  series: [
    { name: 'SOC (kWh)', type: 'line', smooth: true, showSymbol: false, lineStyle: { width: 2 }, data: chartData.value.map((d) => d.soc) },
  ],
  // Reference lines at 0 and usable capacity
  graphic: [],
}))

const barOption = computed(() => ({
  color: ['#22c55e', '#0ea5e9', '#ef4444'],
  animation: false,
  grid: { left: 45, right: 20, top: 20, bottom: 40 },
  textStyle: { color: '#334155' },
  tooltip: { trigger: 'axis' },
  legend: { top: 0 },
  xAxis: { type: 'category', data: chartData.value.slice(0, 120).map((d) => d.day), name: 'Tag 1–120', nameLocation: 'middle', nameGap: 30 },
  yAxis: { type: 'value', name: 'kWh/Tag' },
  series: [
    { name: 'PV', type: 'bar', stack: 'total', data: chartData.value.slice(0, 120).map((d) => d.pv) },
    { name: 'Last', type: 'bar', stack: 'total', data: chartData.value.slice(0, 120).map((d) => d.load) },
    { name: 'Ungedeckt', type: 'bar', stack: 'total', data: chartData.value.slice(0, 120).map((d) => d.unmet) },
  ],
}))
</script>

<template>
  <div class="w-full space-y-6">
    <div class="flex items-center justify-between">
      <h1 class="text-2xl md:text-3xl font-semibold tracking-tight">PV-Autarkie-Simulator</h1>
      <button class="btn btn-neutral" @click="resetDefaults">Zurücksetzen</button>
    </div>

    <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
      <div class="col-span-1 card p-5 space-y-4">
        <div class="text-sm text-slate-600">
          Passe die Parameter an. Das Modell nutzt typische Monatsverteilungen für Norddeutschland und simuliert den Jahresverlauf tagweise.
        </div>

        <UiParam v-model:number="state.kwp" label="PV-Leistung (kWp)" unit="kWp" :min="1" :max="150" :step="1" />
        <UiParam v-model:number="state.specificYield" label="Spezifischer Ertrag" unit="kWh/kWp·a" :min="600" :max="1100" :step="10" />
        <UiParam v-model:number="state.dailyLoad" label="Tagesverbrauch" unit="kWh/Tag" :min="5" :max="120" :step="1" />

        <div class="grid grid-cols-2 gap-3">
          <UiParamSmall v-model:number="state.batteryKwh" label="Batterie (nominal)" unit="kWh" :min="10" :max="2000" :step="10" />
          <UiParamSmall v-model:number="state.usableDoD" label="nutzbare DoD" unit="%" :min="50" :max="100" :step="1" />
        </div>

        <div class="grid grid-cols-2 gap-3">
          <UiParamSmall v-model:number="state.rtEff" label="Roundtrip-Wirkungsgrad" unit="%" :min="70" :max="98" :step="1" />
          <UiParamSmall v-model:number="state.inverterEff" label="PV→AC Wirkungsgrad" unit="%" :min="90" :max="99" :step="1" />
        </div>

        <div class="flex items-center justify-between py-2">
          <label class="flex items-center gap-2">
            <input v-model="state.addRandomness" class="accent-indigo-600" type="checkbox">
            <span class="label">Tagesvariabilität</span>
          </label>
          <div class="flex items-center gap-2">
            <span class="label">Seed</span>
            <input v-model.number="state.randomSeed" class="input w-24" type="number">
            <button class="btn" title="Zufalls-Seed" @click="state.randomSeed = Math.floor(Math.random() * 1e6)">🎲</button>
          </div>
        </div>
      </div>

      <div class="col-span-1 lg:col-span-2 space-y-6">
        <div class="card">
          <div class="p-4">
            <div class="grid grid-cols-2 md:grid-cols-4 gap-4">
              <UiKpi label="Autarkie" :value="`${toFixedNice(sim.autonomy, 1)} %`" />
              <UiKpi label="Jahres-PV" :value="`${toFixedNice(yearPV, 0)} kWh`" />
              <UiKpi label="Batterie nutzbar" :value="`${toFixedNice(usableCap.value, 0)} kWh`" />
              <UiKpi label="Nicht gedeckt" :value="`${toFixedNice(sim.unmetLoad, 0)} kWh`" />
            </div>
          </div>
        </div>

        <div class="card p-4 space-y-3">
          <h2 class="text-lg font-medium">Batterie-SOC über das Jahr</h2>
          <ClientOnly>
            <UiEchart :key="chartKey" :option="socOption" :height="300" />
          </ClientOnly>
        </div>

        <div class="card p-4 space-y-3">
          <h2 class="text-lg font-medium">Tagesbilanz (PV, Last, ungedeckt)</h2>
          <ClientOnly>
            <UiEchart :key="chartKey + '-bar'" :option="barOption" :height="300" />
          </ClientOnly>
          <p class="text-xs text-slate-500">Hinweis: Aus Performancegründen zeigt die Balkengrafik nur die ersten 120 Tage. Passe Parameter an und beobachte die SOC-Kurve fürs Gesamtjahr.</p>
        </div>

        <div class="card p-4 grid grid-cols-1 md:grid-cols-3 gap-4">
          <div>
            <h3 class="font-medium mb-1">Schlimmstes 7-Tage-Fenster</h3>
            <p class="text-sm">Start Tag {{ sim.darkest7.startDay }}, ungedeckt {{ toFixedNice(sim.darkest7.unmet, 1) }} kWh</p>
          </div>
          <div>
            <h3 class="font-medium mb-1">Schlimmstes 14-Tage-Fenster</h3>
            <p class="text-sm">Start Tag {{ sim.darkest14.startDay }}, ungedeckt {{ toFixedNice(sim.darkest14.unmet, 1) }} kWh</p>
          </div>
          <div>
            <h3 class="font-medium mb-1">SOC-Spanne</h3>
            <p class="text-sm">min {{ toFixedNice(sim.minSOC, 1) }} kWh · max {{ toFixedNice(sim.maxSOC, 1) }} kWh</p>
          </div>
        </div>
      </div>
    </div>

    <div class="text-xs text-slate-500">
      Annahmen: Monatsverteilung für Norddeutschland (grobe Richtwerte), tägliche Schwankungen optional. Roundtrip-Wirkungsgrad wird hälftig auf Laden/Entladen verteilt. Keine Selbstentladung/BMS-Standby/Netzverluste, keine Verschattung/Degradation. Ergebnisse sind Näherungen; für Planung bitte standortspezifische Ertragsdaten (PVGIS o.ä.) verwenden.
    </div>
  </div>
 </template>

<style scoped>
input[type="range"] {
  width: 100%;
}
</style>
