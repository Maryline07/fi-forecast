# Financial Independence Forecast

A single-page calculator that projects three scenarios of capital growth over a
user-defined horizon and tells you the year when passive income covers your
target — in today's purchasing power.

## What is it

Enter your current capital, monthly income and expenses, time horizon, and a
target passive income. The page runs three parallel forecasts — **Conservative**
(5 % return / 6 % inflation), **Realistic** (8 % / 5 %), and **Aggressive**
(12 % / 4 %) — and shows:

- the calendar year each scenario reaches financial independence;
- a combined capital-over-time chart (three lines);
- year-by-year tables for each scenario;
- a three-sentence plain-English summary.

No server, no login, no data leaves your browser.

## Demo

**[maryline07.github.io/fi-forecast](https://maryline07.github.io/fi-forecast/)**

## How to run locally

```bash
# Clone and open — that's it.
git clone https://github.com/Maryline07/fi-forecast.git
cd fi-forecast
# On macOS
open index.html
# On Windows
start index.html
# On Linux
xdg-open index.html
```

There is no `npm install`, no build step, no local server required.  
Double-clicking `index.html` in your file manager works too.

## Tech

| Concern | Choice |
|---------|--------|
| Structure | Single `index.html` — HTML, `<style>`, and `<script>` inline |
| Charts | [Chart.js v4](https://www.chartjs.org/) via CDN (`jsdelivr.net`) |
| Fonts | IBM Plex Sans + IBM Plex Mono via Google Fonts |
| Build | None — no npm, no bundler, no transpiler |
| Hosting | GitHub Pages (static, pushed from `main`) |

The entire application is one file you can read, copy, or fork without any
toolchain installed.
