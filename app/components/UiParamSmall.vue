<script setup>
// Support both `v-model:number` and default `v-model` bindings
const props = defineProps({
  label: { type: String, required: true },
  unit: { type: String, required: true },
  min: { type: Number, required: true },
  max: { type: Number, required: true },
  step: { type: Number, required: true },
  number: { type: Number, required: false, default: undefined },
  modelValue: { type: Number, required: false, default: undefined },
})
const emit = defineEmits(['update:number', 'update:modelValue'])

const local = ref(props.number ?? props.modelValue ?? 0)
let tId

watch(
  () => [props.number, props.modelValue],
  () => {
    const incoming = props.number ?? props.modelValue
    if (incoming !== undefined && incoming !== local.value) local.value = incoming
  },
)

function emitBoth(val) {
  emit('update:number', val)
  emit('update:modelValue', val)
}

function onNumberInput(e) {
  const raw = e?.target?.value
  const n = Number(raw)
  local.value = Number.isNaN(n) ? local.value : n
}

function onNumberChange() {
  emitBoth(local.value)
}

function onRangeInput(e) {
  const n = Number(e?.target?.value ?? 0)
  local.value = n
  if (tId) clearTimeout(tId)
  tId = setTimeout(() => emitBoth(n), 60)
}
</script>

<template>
  <div class="space-y-2">
    <label class="label">{{ props.label }}</label>
    <div class="flex items-center gap-2">
  <input v-model.number="local" class="input w-24" type="number" @change="onNumberChange" @input="onNumberInput">
      <span class="text-sm text-slate-500">{{ props.unit }}</span>
    </div>
  <input class="w-full accent-indigo-600" type="range" :min="props.min" :max="props.max" :step="props.step" :value="local" @input="onRangeInput">
  </div>
</template>
