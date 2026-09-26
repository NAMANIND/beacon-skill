---
name: beacon-browser
description: >-
  Automates the user's Chrome through Beacon MCP (observe, navigate, run, click
  by control id). Use when the user invokes /beacon-browser or asks to drive,
  test, or complete flows in their browser from the editor — including app UI,
  third-party sites, forms, and checkout. Not for Playwright, raw DevTools,
  or the Beacon sidepanel chat loop unless explicitly requested.
disable-model-invocation: true
---

# Beacon browser (MCP)

Drive the **user's Chrome** for whatever they asked — app UI in this repo, grocery, flights, admin, signup. Observe → act → verify. When the target is **this** codebase, source + dev terminal matter; otherwise prompt + **Tabs** roster + dump.

Do not start a sidepanel chat session. Do not call `inspect`. Do not use Playwright.

## Run-once pattern (default for multi-step jobs)

**One surface, one MCP `run` per chunk** — not one `click` / `observe` per host round-trip.

1. **Plan** from one dump (`observe` after `navigate`, or `resetFocus: true` on a new page).
2. **`run`** a JSON array of steps on that tab; end the batch with **`expect`** (`line`, `feed`, `url`, or cart badge).
3. **Read `page`** in the `run` response — no standalone `observe` between steps in the same batch.
4. **Next chunk** only after `RUN_FAIL`, `EXPECT_*`, `NOT_FOUND`, `NEEDS_HUMAN`, or a milestone (new search URL, cart overlay, wizard step) → one fresh `observe`, then another **`run`**.

Standalone **`click` / `type`** = recovery or single-action jobs only. **Irreversible** (pay, send, submit search): `commit: "irreversible"` + `expect`.

## Fewer MCP round-trips

Each standalone Beacon tool = **one MCP HTTP round-trip** through the relay, then (in agent hosts) **another model turn**. Chrome work is often ~50ms–1s per step; **serial tools multiply wall time**.

| Slow                                          | Fast                                                  |
| --------------------------------------------- | ----------------------------------------------------- |
| N× `click` + N× `observe` across N host turns | 1× `run` with N steps + one `expect`                  |
| Full `observe()` after each ADD               | Use ids in `run`; read `page` JSON until batch fails  |
| `navigate` between every row on one page      | One navigate per new query; ADDs inside `run`         |
| `observe` while `Ctx tab loading`             | One wait; don’t stack navigates (~10s tab-ready each) |
| Hosted relay when local server is up          | Local MCP url in host config (lower latency)          |
| Retry loops on MCP timeout                    | Two-strike stop (connection errors only)              |

**Budget:** N actions on one tab → **⌈N/8⌉ MCP calls** (batched `run`s). Commerce: one `run` per search results page + `expect` on badge.

## Phased batches (when one `run` is too long)

Split on **milestones** (origin → `expect` → destination → `expect`; or all ADDs on a page → badge `expect`), not per click.

## Tab roster — reuse before open

1. **`observe`** — read `Tab id | url`, **title**, **`Tabs …`**: `*id` = exec tab, `+id` = tabs this job touched.
2. Prompt already matches header url/title → stay; fix path with `navigate` only.
3. Else match **Tabs** titles → `switch_tab` / `navigate` with `title` or `tabId`.
4. **`open_tab` last** — no duplicate site tabs.

**Foreground demo:** Cross-site `navigate` may open a **background** tab. If the user needs the site **visible** (recording), they focus that tab in Chrome first; then reuse it via `navigate` on the same host — prompts alone cannot force Chrome foreground.

## Dump (not HTML)

**Ctx** — respect `navigate` / `hidden` / `re-observe; ids may have changed`. **`Ctx hidden`:** focus exec tab (`navigate({ title: "…" })`) before expecting cart Overlay.

**Scroll** `n## label 42%` = nested scroller id for `scroll`, not the row button. **Feed** is truncated — expand/scroll pane, then `run` ending in `observe`. **template=Item** = row inventory with action ids.

## Gate — only **Beacon dev source** in a workspace

Third-party sites → skip gate; first move is MCP `observe` / `navigate`.

Localhost/vite/next from **a Beacon extension dev clone**: read visible terminal (no shell). Compiling → wait. Never `curl /health` or read `mcp.json` for that.

## MCP — connect once

Auth or list tools **once** when namespace loads. **Auth once per session.** Re-auth does not fix a stuck MCP HTTP stream. Host config url is the relay in use — not necessarily another machine’s `:8787`.

## Health vs hands vs perf

| Signal                                  | Proves                          | Does not prove                       |
| --------------------------------------- | ------------------------------- | ------------------------------------ |
| `/health` `{ ok: true }`                | Relay process up                | Extension linked, commands delivered |
| MCP client “connected”                  | HTTP session                    | Next tool reaches Chrome             |
| `[beacon perf] tool Nms` (extension SW) | That command finished in Chrome | Other MCP calls                      |

No perf line for the failing call → break above extension (MCP client / relay queue), not page slowness. Do not `curl /health` mid-task.

### Forbidden when MCP fails (unless fixing beacon-server install)

No `curl /health`, read `mcp.json`, search `beacon-server`, `npm run dev` on 8787, shell probe loops, or “disconnected” without a failed **`observe`** in this session.

## Start every job with one `observe`

Browser task in this chat → **call `observe` first** (or `navigate` then `observe`). If it returns a dump, **Chrome is reachable** — do not claim “can’t reach Chrome” or “relay isn’t talking to the extension” later in the same turn.

## Tool failures — two strikes (MCP connection only)

**Strike counter:** only full-string **`Timed out waiting for connection`** on a Beacon MCP tool. Count them in **this chat**:

| Count | Move                                                                                                        |
| ----- | ----------------------------------------------------------------------------------------------------------- |
| 0     | User asked for browser work → **`observe`** now                                                             |
| 1     | Retry **once** (`observe` or repeat the same call). No essays.                                              |
| 2     | Short stop: exact error twice, last url if any, “reconnect MCP or new chat”. **No substitute deliverable.** |

Not a strike: `RUN_FAIL`, `EXPECT_*`, `NOT_FOUND`, `BLOCKED`, `browser timeout`, one slow call, namespace `loading`.

**Do not quit the job on strike 1.** One failed ADD ≠ disconnect — **`run` again**.

### Forbidden substitutes (when the user asked you to _do_ it in Chrome)

Do **not** replace automation with:

- Shopping / meal / party **plans**, SKU tables, or budget spreadsheets
- Long **“Reconnect Beacon”** playbooks (max 2 lines at strike 2)
- **“Per the Beacon workflow I’m stopping”** without two connection timeouts quoted from this chat
- **“I did not add anything to the cart”** if you never got a successful `observe`/`run` in this chat — say “tools failed before first observe” instead
- Closing with **Decide / retry when fixed** while MCP might work on the next `observe` in a **new** chat

If the user wanted a cart built, **building the cart** (or honest strike 2 after two timeouts) is the only acceptable outcome — not a planning doc.

### After a failed `run` step (keep going)

1. Read `log` / `page` — what succeeded (badge 0→1 items?).
2. **`observe`** (`resetFocus: true` if Ctx says spa/hidden).
3. **`run`** remaining steps with **fresh ids**.
4. Until budget/milestone, `NEEDS_HUMAN`, or MCP strike 2.

Partial cart + open search = **continue**, not a handoff.

## Commerce (cart, checkout)

- **Whole job:** budget (e.g. ₹1500) until done — not one SKU then quit.
- One **`run`** per results page + `expect` on badge; mid-batch fail → observe → **`run`** rest.
- Badge: **`n## button N items`** / **`N items ₹…`**. `/cart` often stays home — badge + **Overlay**.
- No **pay** without line items + total (`commit: irreversible` + `expect`).
- One SKU without id → skip, observe, continue.

## Serial state (lists, wizards, flights)

One tab. **No parallel Beacon calls** on one job. Don’t re-`navigate` the same URL to “reset” rows. One expanded panel at a time; use **`run`** with `observe` inside the batch when UI shifts. `BLOCKED` on price link → use the named button id from **template=Item**.

## Fast path (target is this repo)

**One control:** `navigate` → `observe` → `click`/`type` that `n##`.

**Several:** one `observe` → one **`run`** + `expect`.

## Hard stops

- Id first. No name-only clicks. No cousin clicks. Latest dump ids only.
- **Forbidden close:** “stopped responding” / “reconnect MCP” when last MCP call **succeeded** or only `run` step failed.
- **Forbidden:** `Couldn't reach Chrome` header + shopping plan — that pattern is always wrong unless strike 2 with two quoted timeout errors and zero successful `observe` in the session.

## Tools

Primary: **`run`**. Support: `observe`, `navigate`, `scroll`, `press`, `open_tab`, `switch_tab`. `click` / `type` for single-step or after failed `run`.

`observe`: `{ region }` / `{ id }`; `resetFocus: true` after navigate. `run`: `commit` **`reversible`** | **`irreversible`** only.

## What this skill does not fix

- MCP transport stalls (no `[beacon perf]` for that call) — strike 2; human reconnects host MCP.
- Missing ids on a live site — extension/eval; skip line and continue where possible.
- Irreversible gates — still require `expect`.

## Setup (humans only — not agent recovery)

Install from the **beacon-skill** repo README. Wire Beacon MCP + extension at [trybeacon.space/setup](https://www.trybeacon.space/setup). Agents: successful `observe` = hands work.
