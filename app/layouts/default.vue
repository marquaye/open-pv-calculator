<template>
  <div class="min-h-screen flex flex-col bg-gradient-to-b from-slate-50 to-white text-slate-800">
    <header class="sticky top-0 z-30 backdrop-blur supports-[backdrop-filter]:bg-white/70 bg-white/90 border-b">
      <div class="max-w-6xl mx-auto px-4 py-3 flex items-center justify-between">
        <NuxtLink to="/" class="flex items-center gap-2 font-semibold tracking-tight">
          <Icon name="ph:solar-panel" size="22" />
          <span>Open PV Calculator</span>
        </NuxtLink>
        <nav class="text-sm text-slate-600 flex items-center gap-4">
          <NuxtLink to="/pv-simulator" class="hover:text-slate-900">Simulator</NuxtLink>
          <a href="https://github.com/marquaye/open-pv-calculator" target="_blank" class="hover:text-slate-900">GitHub</a>
        </nav>
      </div>
    </header>
    <main class="flex-1 max-w-6xl w-full mx-auto px-4 py-6">
      <slot />
    </main>
  <footer class="border-t text-xs text-slate-500">
      <div class="max-w-6xl mx-auto px-4 py-4">MIT © {{ new Date().getFullYear() }}</div>
    </footer>
  </div>
</template>

<style>
:root { font-family: Inter, ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, "Apple Color Emoji", "Segoe UI Emoji"; }
</style>
