# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A **single-file, static, build-less web app**: `industry-map.html` is the entire application — HTML, CSS, and one `<script>` of vanilla JavaScript. There is no backend, no `package.json`, no build step, and no test framework. The codebase and UI are in **Chinese**.

`index.html` is only a redirect shim (`location.replace('industry-map.html')`) so the GitHub Pages root URL lands on the app. All real work happens in `industry-map.html`.

## Running & developing

- **Run it**: open the file in a browser — `start industry-map.html` (Windows). ECharts loads from a CDN, so the first open needs internet.
- **Edit loop**: change `industry-map.html`, reload the browser. No watcher/server needed (a server isn't required because there's no module loading — everything is inline).
- **Validate inline JS after editing** (there is no linter; this is the syntax check used in this repo):
  ```bash
  node -e 'const fs=require("fs"),vm=require("vm");const h=fs.readFileSync("industry-map.html","utf8");for(const m of h.matchAll(/<script>([\s\S]*?)<\/script>/g)){const c=m[1].trim();if(c)new vm.Script(c);}console.log("JS OK")'
  ```
- **Deploy**: `git push origin main` → GitHub Pages rebuilds in ~1 min. Live at https://beta23690-arch.github.io/global-industry-map/ . `.claude/` is gitignored.

## Architecture of `industry-map.html`

The single `<script>` is organized top-to-bottom into these regions (search the `/* ===== ... ===== */` banners):

1. **Content model — `DATA`** (~line 175): an array of 12 industries, each `{ id, name, color, intro, fields:[{ name, companies:[{ name, ticker, loc, desc }] }] }`. This is the source of truth for every node, accordion, and company card.
2. **Cross-industry edges — `LINKS`** (~line 465): `{s, t, r}` = source id, target id, relation label. These become the labeled lines between industry nodes.
3. **Chart rendering** (~line 485): `DATA`/`LINKS` are transformed into ECharts `nodes`/`links` (node size scales with field + company counts) and rendered as a force-directed `graph`. Clicking a node calls `openPanel(id)`.
4. **Detail panel** (~line 530): `openPanel` renders an **industry-overview card** (`#p-overview`) + the field *shells* (empty `.field-inner`) and stashes the industry in `curD`, then fires `loadIndustry` — one **batched** quote request (`fetchTencentBatch`) for the whole industry that fills `quoteCache` and computes the overview (total USD mcap, median PE, median 52-week position). `openPanel` also renders a **sort/filter control bar** (`controlsHTML`) under the overview. Expanding a field calls `renderField` (lazy, but also re-run on control change): reads quotes from the warmed cache (falls back to per-company fetch), applies the global `marketFilter` (`all`/`us`/`cn`/`hk`, via `marketOf(sym)`) and sorts by the global `sortKey` (`mcap` desc / `pe` asc / `year` desc / `day` desc — see `cmpRows`; `year` needs a kline fetch so it's awaited only for that mode), builds each card via `cardHTML` — name/ticker/price badge, a market-cap size bar, a metrics row (市值 / PE / 52-week position), **and a one-year sparkline** — then `loadSparklines` fills each sparkline async. `setSort`/`setMarket` update the module state and call `reRenderOpenFields`. Companies with no quote (未上市, no mapping) sort to the bottom and show `—`. **Compare**: each card has a `＋` toggle (`toggleCompare`) that adds its sym to the module `compareSet` (max 3); selected cards highlight (`.sel`) and show in a fixed bottom tray (`renderTray` → `#cmp-tray`). `openCompare` builds a side-by-side modal (`#cmp-modal`) with a metrics table (现价/当日/近一年/走势/市值/PE/52周位置), reusing the cached quote + kline. Works across industries because `compareSet`/`symInfo` are global; Esc closes the modal before the panel.
5. **Real-time quotes** (~line 572 onward) — see below.

### The quote system (the non-obvious part)

**One data source for everything: tencent `qt.gtimg.cn` — no API key.** `fetchTencent` calls `https://qt.gtimg.cn/q=<sym>` directly (the endpoint returns `CORS: *`), decodes the response as **GBK**, splits on `~`, and reads `a[3]`=current price, `a[4]`=prev close (`parseTencent`). The non-obvious bit: gtimg quotes **US stocks and overseas ADRs too**, via a `us` prefix (`usAAPL`, `usNSRGY`, `usBRK.B`) — all USD. So a single keyless source covers US + ADR + A-share + HK.

`quoteSym(ticker)` is the router that turns a `DATA` ticker into a gtimg symbol, checking three tables in order:

- **`TENCENT`** (`ticker` → `sh601398`/`hk00700`): A-shares (¥) / HK (HK$), tencent-native codes.
- **`QSYM_OVR`** (`ticker` → full gtimg code): one-off exceptions whose home exchange gtimg can't quote, e.g. Samsung `005930` → `ukSMSN` (its London GDR, USD, live-moving).
- **`QSYM`** (`ticker` → US/ADR symbol, *without* the `us` prefix): US stocks and overseas ADRs. `quoteSym` prepends `us` (`NVDA`→`usNVDA`, `NESN`→`usNSRGY`). Most European/Japanese names quote through their US OTC ADR here.

A ticker matching none of the three → no `data-sym` attr → card shows `—` (e.g. Saudi Aramco `2222`, and all 未上市 names). `parseTencent` returns not just price/change but also **`pe`, `mcap` (local 亿), `mcapUsd`, `hi52`, `lo52`** — pulled from market-specific field indices (US/ADR & HK vs A-share differ by one; the `uk` Samsung GDR has unreliable valuation fields so they're **skipped**). `mcapUsd = mcap × FX_USD[cur]` (static approx rates — HKD pegged, CNY stable) so caps are comparable across currencies for **sorting, the size bar, and display** (non-USD shown as `≈$…` via `fmtMcapUsd`). `companyURL()` routes the card click-through (US/ADR → Yahoo Finance, A/HK → 东方财富, `QSYM_OVR` 6-digit → Yahoo home listing e.g. `005930.KS`, unlisted → Bing). Price-badge currency symbol per market via `CUR_SYM`.

**To add a company with a working live price you must do two things**: add it to `DATA`, *and* add its `ticker`→symbol entry to the right table (`QSYM` for US/ADR, `TENCENT` for A/HK, `QSYM_OVR` for an exotic exception). Quick way to find a working symbol: `node -e "fetch('https://qt.gtimg.cn/q=usTICKER').then(r=>r.arrayBuffer()).then(b=>console.log(new TextDecoder('gbk').decode(b)))"` — non-empty `v_...="200~..."` means it quotes. Without a mapping the card renders but the price shows `—`.

**Sparklines (one-year mini charts)** also come from tencent — `fetchKline` hits `web.ifzq.gtimg.cn/appstock/app/fqkline/get`. A-shares/HK use the plain symbol (`sh601398`); US/ADR need the **full exchange code** (`us` + `q.code`, e.g. `usAAPL.OQ`, `usBRK.B.N`) where `q.code` is `a[2]` from the quote. (Eastmoney `push2his` also serves these but was dropped — it's a mainland-only host and flaky from outside China; tencent is a global CDN.) Cached in `klineCache`, never refetched on the 60s tick. Only Samsung's GDR (`ukSMSN`) has no kline history, so it's the one listed name without a sparkline.

### Refresh & flash

`setInterval(refreshOpenQuotes, 60000)` refreshes only **expanded** fields (`.field.open`): per `.company[data-sym]` it clears `quoteCache`, re-fetches, and updates badge + metrics **in place** via `applyCardQuote` (no re-sort, so cards don't jump). `markRefreshed()` updates the panel's "已于 HH:MM:SS 刷新" status bar each cycle (proof the timer ran, even when markets are closed and prices don't move). `lastPrice` holds the previously shown price per symbol so `applyCardQuote` can flash a quote green/red on change (no flash on first load).

## Constraints & gotchas

- **Unofficial endpoints**: tencent (`qt.gtimg.cn`) and 东方财富 are public but undocumented — free, no key, but no SLA and may break on their redesign. Personal/educational use only.
- **Sina (`hq.sinajs.cn`) is deliberately excluded**: since 2022 it enforces a `Referer` check that a pure front-end page can't satisfy. Don't reintroduce it.
- **ECharts is CDN-loaded**, so the app is not fully offline-capable as-is.
- Quote calls are cross-origin `fetch`; this works because gtimg sends a permissive `CORS: *` header. Keep new data sources to ones that do the same, or they'll fail silently to `—`. (A prior Finnhub-key path for US stocks was removed once gtimg's `us`-prefix was found to cover US/ADR keylessly — don't reintroduce a key requirement.)
