# Open PV Calculator

An open-source PV autonomy and sizing simulator with a simple cost model. Built with Nuxt 4, Vue 3, ECharts, and Tailwind CSS.

The main simulator lives in `app/pages/pv-simulator.vue` and focuses on off-grid/backup scenarios: how much PV and battery are needed to cover a given daily load through the year.

## Features

- Interactive PV autonomy simulator (German UI)
	- Inputs: PV size (kWp), specific yield (kWh/kWp·a), daily load (kWh/day), battery size (kWh), usable DoD, roundtrip and inverter efficiency, randomness toggle and seed
	- Outputs: Autarkie (self-sufficiency), annual PV energy, usable battery capacity, unmet energy, SOC curve over a year, daily PV/Load/Unmet (first 120 days), worst 7- and 14-day deficit windows, SOC min/max
- Cost estimate (rule-of-thumb)
	- Panels: 450 W modules at $50 each, +5% for wiring (module count rounded up)
	- Battery: cells at $50/kWh, +5% for enclosure and BMS
	- Total shown as USD; tweak constants in `pv-simulator.vue` to change currency/prices
- Fast and lightweight UI using ECharts and Tailwind

## How it works (model overview)

- Annual PV = kWp × specific yield; distributed across months using a typical profile for Northern Germany
- Daily PV variability with a seeded RNG, stronger in winter months
- Battery is modeled with a roundtrip efficiency split evenly across charge/discharge (√η); separate inverter efficiency on PV → AC
- Initial SOC = 50% of usable capacity; no self-discharge; no degradation; no standby loads
- Rolling-window analysis highlights worst 7- and 14-day unmet-energy windows

These are engineering approximations; use site-specific data (e.g., PVGIS) for real planning.

## Quick start

Prerequisites: Node.js 18+ (or Bun), a package manager of your choice.

Install dependencies:

```bash
# npm
npm install
# pnpm
pnpm install
# yarn
yarn install
# bun
bun install
```

Run the dev server (http://localhost:3000):

```bash
# npm
npm run dev
# pnpm
pnpm dev
# yarn
yarn dev
# bun
bun run dev
```

## Build & deploy

Build for production:

```bash
# npm
npm run build
# pnpm
pnpm build
# yarn
yarn build
# bun
bun run build
```

Preview the production build locally:

```bash
# npm
npm run preview
# pnpm
pnpm preview
# yarn
yarn preview
# bun
bun run preview
```

Static generation is available via:

```bash
# npm
npm run generate
```

See Nuxt’s [deployment docs](https://nuxt.com/docs/getting-started/deployment) for platform-specific steps (Vercel, Netlify, Static Hosting, etc.).

## Tech stack

- Nuxt 4 (Vue 3)
- ECharts for charts (`UiEchart` wrapper component)
- Tailwind CSS

## Project layout

- `app/pages/pv-simulator.vue` – main simulator page and logic
- `app/components/` – reusable UI bits (params, KPIs, charts)
- `nuxt.config.ts` – Nuxt configuration
- `tailwind.config.ts` / `assets/css/tailwind.css` – styling

## Cost model details

In `app/pages/pv-simulator.vue` you can adjust these constants:

- 450 W per panel at $50 each, +5% for wiring
- Battery cells at $50/kWh, +5% for enclosure and BMS
- Panel count = ceil(kWp × 1000 / 450)

Change prices, markups, or currency formatting as needed.

## Contributing

Issues and PRs are welcome. To work locally:

1. Fork and clone the repo
2. Install deps and run the dev server
3. Make focused changes with clear commit messages
4. Open a PR with a concise description and screenshots if UI changes

## Localization

The UI labels are currently in German. Contributions for i18n are welcome.

## Disclaimer

This tool provides approximate results for educational and early sizing purposes. It does not replace detailed, site-specific engineering.

## License

Please add a LICENSE file to declare your chosen license (e.g., MIT). If you’d like, this project can include an MIT license by default.
