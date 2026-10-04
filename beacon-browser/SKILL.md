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

## One `run`, expect on every step

Put the whole known flow in one `run` with an `expect` on each step (origin line, destination line, date, results). A run of 2+ steps with any bare step is rejected (`observe`, `finish`, and `mode` are exempt). Use `ref` for overlay options that appear mid-flow. On a miss, re-plan from the returned page — don't observe first.

## Tab roster — reuse before open

1. **`observe`** — read `Tab id | url`, **title**, **`Tabs …`**: `*id` = this lane's work tab, `+id` = other tabs Beacon opened for this lane (tabs it only adopted from the user get no mark). `Work URL:` is the lane's tab, not necessarily the tab the user is looking at.
2. Prompt already matches header url/title → stay; fix path with `navigate` only.
3. Else match **Tabs** titles → `navigate` with `title` or `tabId`.
4. **New tab last** (`navigate` with `newTab`, or a `run` step `open_tab`) — no duplicate site tabs.

## Dump (not HTML)

**Ids** look like `n29kq7`: a number plus a 3-character tag naming the page that printed it (`Overlay/n29kq7` when region-scoped; frames look like `n7000012kq7`). Copy ids exactly; never strip, invent, or carry the tag across pages. A dump's ids are valid only on that page until it navigates or reloads.

**Ctx** — respect `navigate` / `hidden` / `wait before acting`. **`Ctx hidden`:** focus exec tab (`navigate({ title: "…" })`) before expecting cart Overlay. **`observe region`** accepts any name the dump prints: a `Regions` entry or a `Form <name>` (`region: "form"`).

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

## Lanes (several tasks in one Chrome profile)

Other chats may drive the same profile at the same time. **Pick one short lane name per task** (`flights`, `gmail`, `sheet-q3`) and pass **`lane`** on **every** call — `observe`, `navigate`, `run`, `click`, all of it. Same name for the whole task; never reuse another task's name. Omitted lane = `default` (the user's active tab; only for a single job).

A non-default lane (and **agent** mode on `default`) never touches the user's tab. No empty tab is opened up front — the first **`navigate` / `open_tab`** with a URL creates the lane's background tab. **A lane is one agent's private track:** its own queue, tabs, page ids, and **its own Chrome tab group** titled with the lane name (name it by role, e.g. `Mail agent`, `Flight agent`; the user reads it on the group), in a color no other live lane has. Steps inside a lane run one at a time; use more lanes for work that must run at the same moment. Any number of agents can share a profile; at most **5** lanes run at once per profile and the rest wait their turn (~90s). **A lane idle for 10 minutes is finished automatically** (its tabs close) unless the user stopped it or is looking at one of its tabs; after that, `navigate` again to start fresh. **Cleanup is one `run` step, not an MCP tool:** end every job with `{"tool":"finish"}` in a `run` carrying your `lane`. It closes the tabs Beacon opened for that lane, releases it, and removes the tab group once no lane has tabs left. A `run` with no `lane` tears down every lane on the profile, so only do that when no other chat is using it. Tabs the user opened and the tab they are looking at are never closed; when `finish` says `Kept tab N open`, tell the user that tab stayed and why. If the user asks to leave the tabs open ("keep it open", "leave the results up"), end with `{"tool":"finish","args":{"keep":true}}`: the lane is released and ungrouped, and its tabs stay open as the user's. Do not use `close_tab` for teardown.

**Mode** is per Chrome profile and shown on every dump: `Browser @work · agent` or `Browser @work · personal`. In **agent** mode every task, including calls with no `lane`, runs in background tabs and never takes the user's active tab; `focus` is ignored; on `NEEDS_HUMAN` the tab is brought to the front for the user — tell them what to do, then wait. In **personal** mode (default) the `default` lane works in the user's current tab. Read it from the Browser line. Change it only when the user asks, with a `run` step: `{"tool":"mode","args":{"set":"agent"}}` or `"personal"` (add `"browser":"work"` on the step when several profiles are online). It applies before the other steps in that run; the side panel toggle uses the same setting. Hosted chats can't change it. Tabs you opened stay open until the user presses Reset on the Board; close them yourself with `close_tab` when the result no longer needs them. Discord / X composers: **`type` → `focus` composer id → `press` `Return`** as steps of one `run`; bare `Enter` without focus can return `NOT_SENT`.

**Main profile:** the server marks the first Chrome profile linked to an account as primary; nobody can change it. MCP `browser: "primary"` (alias `main`) routes to that profile when it is online. If it is revoked, the oldest remaining profile becomes primary.

**Multi-profile:** when the header shows `also: @other`, pass **`browser` on every tool including `run`**. Omitted `browser` on a **new lane** → `AMBIGUOUS_BROWSER`. On a lane that already observed, the server picks the profile that produced that lane's last dump (not a stale MCP sticky from another chat). Cross-profile id misuse → **`STALE`** even when standalone act tools are disabled. **Discord:** `navigate` to the full `discord.com/channels/<guild>/<channel>` URL — do not guess channel names.

| Error               | Meaning                                                                         | Move                                                       |
| ------------------- | ------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| `NO_INJECTABLE_TAB` | Isolated lane has no owned https tab yet                                        | `navigate` or `open_tab` with a URL first.                 |
| `WRONG_LANE`        | That tab belongs to another lane, or is the user's own tab                      | Do not switch to it. `navigate` / `open_tab` in your lane. |
| `LANE_STOPPED`      | The user pressed Stop or Reset on the Agent Board                               | Stop the task and tell the user. Do not retry or rename.   |
| `BUSY`              | 5 lanes busy and no slot opened within ~90s (server waited in-memory)             | Retry with the **same** lane name; do not invent a 6th lane. |
| `STALE` (lane)      | `STALE: observe in lane X first` — ids came from another lane's or profile's dump | `observe` in your lane, then act on the fresh ids.         |
| `STALE` (page)      | `STALE: id n29kq7 is from another page; observe first` — the id's page tag is not this tab's current document (another tab, another lane, or the page navigated/reloaded since that dump). Nothing was clicked. | `observe`, then use the new ids. In a `run`, ids taken before a navigating step are always `STALE`; use `ref` for steps after a navigation. |
| `AMBIGUOUS_BROWSER` | Several profiles online and this lane has no prior observe (or you omitted `browser` on a fresh lane) | Pass `browser` every call when `also: @…` appears — label from the header, or `primary` / `main` (`default` is a lane name, never a profile). |

## Serial state (lists, wizards, flights)

One tab per lane. **No parallel Beacon calls** within one lane. Don’t re-`navigate` the same URL to “reset” rows. One expanded panel at a time; use **`run`** with `observe` inside the batch when UI shifts. `BLOCKED` on price link → use the named button id from **template=Item**.

## Fast path (target is this repo)

**One control:** `navigate` → `observe` → `click`/`type` that `n##`.

**Several:** one `observe` → one **`run`** + `expect`.

## Hard stops

- Id first. No name-only clicks. No cousin clicks. Latest dump ids only.
- **Forbidden close:** “stopped responding” / “reconnect MCP” when last MCP call **succeeded** or only `run` step failed.
- **Forbidden:** `Couldn't reach Chrome` header + shopping plan — that pattern is always wrong unless strike 2 with two quoted timeout errors and zero successful `observe` in the session.

## Tools

Primary: **`run`**. Every action goes inside a `run` step — `click`, `type`, `press`, `scroll`, `focus`, `open_tab`, `switch_tab`, `finish`, `mode`. Standalone MCP tools: `observe`, `navigate` (also switches tabs with `tabId` or `title`), `feedback`. Standalone `click` / `type` / `press` / `scroll` / `focus` return `SERIAL_ACT_DISABLED` on the hosted relay; put them in a one-step `run` instead.

`observe`: `{ region }` / `{ id }`; `resetFocus: true` after navigate. `run`: `commit` **`reversible`** | **`irreversible`** only.

## What this skill does not fix

- MCP transport stalls (no `[beacon perf]` for that call) — strike 2; human reconnects host MCP.
- Missing ids on a live site — extension/eval; skip line and continue where possible.
- Irreversible gates — still require `expect`.

## Setup (humans only — not agent recovery)

Copy this folder to the host skills path (see repo README). Wire Beacon MCP + extension. Agents: successful `observe` = hands work.
