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
4. **Detail panel** (~line 530): `openPanel` builds the slide-out HTML from `DATA`; `toggleField` expands an accordion field and triggers `loadQuotes` for the companies in it.
5. **Real-time quotes** (~line 572 onward) — see below.

### The quote system (the non-obvious part)

**One data source for everything: tencent `qt.gtimg.cn` — no API key.** `fetchTencent` calls `https://qt.gtimg.cn/q=<sym>` directly (the endpoint returns `CORS: *`), decodes the response as **GBK**, splits on `~`, and reads `a[3]`=current price, `a[4]`=prev close (`parseTencent`). The non-obvious bit: gtimg quotes **US stocks and overseas ADRs too**, via a `us` prefix (`usAAPL`, `usNSRGY`, `usBRK.B`) — all USD. So a single keyless source covers US + ADR + A-share + HK.

`quoteSym(ticker)` is the router that turns a `DATA` ticker into a gtimg symbol, checking three tables in order:

- **`TENCENT`** (`ticker` → `sh601398`/`hk00700`): A-shares (¥) / HK (HK$), tencent-native codes.
- **`QSYM_OVR`** (`ticker` → full gtimg code): one-off exceptions whose home exchange gtimg can't quote, e.g. Samsung `005930` → `ukSMSN` (its London GDR, USD, live-moving).
- **`QSYM`** (`ticker` → US/ADR symbol, *without* the `us` prefix): US stocks and overseas ADRs. `quoteSym` prepends `us` (`NVDA`→`usNVDA`, `NESN`→`usNSRGY`). Most European/Japanese names quote through their US OTC ADR here.

A ticker matching none of the three → no `data-sym` attr → price shows `—` (e.g. Saudi Aramco `2222`, and all 未上市 names). `parseTencent` sets currency from the symbol prefix: `us`/`uk`→USD, `hk`→HKD, else CNY. `loadQuotes` reads each `.quote` element's `data-sym` and fetches via the single tencent path. `companyURL()` routes the card click-through (US/ADR → Yahoo Finance, A/HK → 东方财富, `QSYM_OVR` 6-digit → Yahoo home listing e.g. `005930.KS`, unlisted → Bing). Currency symbol per market via `CUR_SYM`.

**To add a company with a working live price you must do two things**: add it to `DATA`, *and* add its `ticker`→symbol entry to the right table (`QSYM` for US/ADR, `TENCENT` for A/HK, `QSYM_OVR` for an exotic exception). Quick way to find a working symbol: `node -e "fetch('https://qt.gtimg.cn/q=usTICKER').then(r=>r.arrayBuffer()).then(b=>console.log(new TextDecoder('gbk').decode(b)))"` — non-empty `v_...="200~..."` means it quotes. Without a mapping the card renders but the price shows `—`.

### Refresh & flash

`setInterval(refreshOpenQuotes, 60000)` refreshes only **expanded** fields (`.field.open`): it clears `quoteCache` and re-fetches. `markRefreshed()` updates the panel's "已于 HH:MM:SS 刷新" status bar each cycle (proof the timer ran, even when markets are closed and prices don't move). `lastPrice` holds the previously shown price per symbol so `applyQuote` can flash a quote green/red on change (no flash on first load).

## Constraints & gotchas

- **Unofficial endpoints**: tencent (`qt.gtimg.cn`) and 东方财富 are public but undocumented — free, no key, but no SLA and may break on their redesign. Personal/educational use only.
- **Sina (`hq.sinajs.cn`) is deliberately excluded**: since 2022 it enforces a `Referer` check that a pure front-end page can't satisfy. Don't reintroduce it.
- **ECharts is CDN-loaded**, so the app is not fully offline-capable as-is.
- Quote calls are cross-origin `fetch`; this works because gtimg sends a permissive `CORS: *` header. Keep new data sources to ones that do the same, or they'll fail silently to `—`. (A prior Finnhub-key path for US stocks was removed once gtimg's `us`-prefix was found to cover US/ADR keylessly — don't reintroduce a key requirement.)
