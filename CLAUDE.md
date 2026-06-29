# Temperaturrechner

A single-file, client-side web app that estimates the steady-state case
temperature of an electronic control unit (ECU) enclosure in an electric
vehicle, based on a simplified thermal-resistance model. German-language UI.

## Repository structure

The entire application lives in **`index.html`** — there is no build step,
no package manager, and no other source files:

- `<style>` block: all CSS (CSS custom properties for colors, mobile-first
  layout, iOS-style cards/tabs).
- `<body>`: four tabs (`Eingabe` / `Ergebnisse` / `Diagramm` / `Info`) toggled
  via `setActiveTab(index)`.
- `<script>` block: all application logic (material data, physics
  calculations, DOM updates, Chart.js wiring).
- External dependency: [Chart.js 3.7.1](https://cdnjs.cloudflare.com/ajax/libs/Chart.js/3.7.1/chart.min.js)
  loaded from a CDN — no local copy, no `node_modules`.

There is no test suite, linter, or CI configuration in this repo.

## Running / developing

Just open `index.html` in a browser, or serve the directory with any static
file server (e.g. `python3 -m http.server`) so relative paths and the CDN
script resolve correctly. Reload the page after edits — there's no hot
reload.

To verify a change visually, use the `/run` skill or launch a browser and
exercise the **Eingabe** tab (inputs auto-recalculate on `input` events),
then check **Ergebnisse** and **Diagramm**.

## Domain model (read before changing calculations)

The core calculation in `calculateTemperature()` models heat transfer as a
simple series thermal-resistance circuit:

1. **Inputs** (all read from the `Eingabe` tab, IDs match variable names):
   `powerLoss` (W), `material`, `length`/`width`/`height`/`thickness` (mm),
   `ambientTemp`/`maxTempWarning` (°C), `convectionType`, `powerDistribution`,
   `surroundings`.
2. Dimensions are converted mm → m, then enclosure `surfaceArea` is computed
   as a closed box: `2*(l*w + l*h + w*h)`.
3. Convection coefficient is a fixed constant: `NATURAL_CONV_COEFF = 5` or
   `FORCED_CONV_COEFF = 25` W/(m²·K), selected by `convectionType`.
4. Thermal resistances: `convectionResistance = 1/(h*A)`,
   `materialResistance = thickness/(k*A)` where `k` comes from `MATERIALS`.
5. `temperatureIncrease = powerLoss * (convectionResistance + materialResistance)`;
   `caseTemperature = ambientTemp + temperatureIncrease`.
6. `MATERIALS` (`Aluminium`, `PVC`) define `thermalConductivity`,
   `density`, `specificHeat`, `emissivity`, and a chart `color`. Radiation
   (Stefan-Boltzmann, `STEFAN_BOLTZMANN`) and `surroundings`/
   `powerDistribution` are referenced but only `powerDistribution` actually
   affects output today — it skews the 5×5 `tempMap` heatmap
   (`calculateTemperatureDistribution`) toward a hotter center for
   `punktuell`, otherwise the grid is uniform.
7. `calculateComparisonData()` recomputes case temperature for both
   materials across a fixed `ambientTempRange` ([25..55] in steps of 5) to
   populate the comparison table and feed `temperatureChart`.

When changing the physics, keep `calculateTemperature`,
`calculateComparisonData`, and `calculateTemperatureDistribution` consistent
— they duplicate the resistance formula rather than sharing one function.

## Conventions

- **Language**: all UI text, labels, and comments are German. Keep new
  user-facing strings in German to match existing tone.
- **IDs as the wiring**: `inputElements` (array of element IDs) drives event
  listener setup in `setupEventListeners()`. When adding a new input field,
  add its `id` to both the HTML and the `inputElements` array, or it won't
  trigger recalculation.
- **No frameworks**: vanilla DOM APIs only (`getElementById`,
  `createElement`, `classList`), consistent with the rest of the file. Don't
  introduce a framework or bundler for a single-page calculator like this.
- **Recalculation flow**: every input change calls `calculateTemperature()`,
  which fans out to `updateResults()`, `calculateTemperatureDistribution()`,
  and `calculateComparisonData()` (which itself calls `updateChart()`). Keep
  this single entry point if you add new outputs.
