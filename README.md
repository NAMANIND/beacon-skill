# beacon-browser skill

## What is Beacon?

[Beacon](https://www.trybeacon.space) is a Chrome extension plus MCP relay that turns **your logged-in browser** into something agents can drive accurately. It **observes** each page as compact text where every clickable control has a stable **`n##` id**, then **clicks**, **types**, and **runs** batched steps against those ids — not screenshots, not guessed selectors, not a separate cloud browser. Sign up, link Chrome, and paste one MCP config into Cursor, Claude Code, or any MCP client.

**This repo** is the **agent skill** (`/beacon-browser`): how models should use those tools (batch `run`, id-first acts, verify before pay). Extension install and keys live on [trybeacon.space/setup](https://www.trybeacon.space/setup).

## Prerequisites

1. [Beacon setup](https://www.trybeacon.space/setup) — Chrome extension + MCP key (`Bearer bcn_…`).
2. MCP server in your agent host pointing at `https://api.trybeacon.space/mcp` (or your relay URL) with that bearer.
3. Successful **`observe`** in a chat = Chrome is wired; if you get **401**, mint a new key on setup and update MCP config + extension.

## Install

Copy the skill folder into the path your host reads:

| Host | Path |
|------|------|
| **Cursor** | `~/.cursor/skills/beacon-browser/SKILL.md` |
| **Claude Code** | `~/.claude/skills/beacon-browser/SKILL.md` |

```bash
git clone https://github.com/NAMANIND/beacon-skill.git
mkdir -p ~/.cursor/skills/beacon-browser ~/.claude/skills/beacon-browser
cp beacon-skill/beacon-browser/SKILL.md ~/.cursor/skills/beacon-browser/
cp beacon-skill/beacon-browser/SKILL.md ~/.claude/skills/beacon-browser/
```

Restart the agent or start a new session so skills reload.

## Use

Invoke **`/beacon-browser`** (this skill sets `disable-model-invocation: true`, so slash it or attach the skill explicitly).

Example prompt:

```text
/beacon-browser

On Blinkit, build a party cart for ~6 under ₹1500 (snacks, drinks, decorations, cutlery).
Stop before pay and show the cart total.
```

### Live demo (visible tab)

Cross-site **`navigate`** may open a **background** tab so another window can stay on screen. For a recording, **focus Blinkit (or your target site) in Chrome first**, then ask the agent to run on the **active tab** and reuse it — do not rely on the prompt alone to foreground Chrome.

## What the skill teaches agents

- Batch work with MCP **`run`** + **`expect`**, not serial click/observe loops.
- **Id-first** clicks from the latest observe dump.
- Commerce, tab roster, MCP strike rules, and no fake shopping lists when the user asked for a real cart.

Tools: `observe`, `navigate`, `run`, `click`, `type`, `press`, `scroll`, `focus`, `feedback`.

## FAQ

**What’s the difference between Beacon and this skill?**  
Beacon is the product (extension + MCP + relay). This repo is copy-paste instructions so agents use Beacon correctly. You need both for best results.

**`/beacon-browser` or skill not listed**  
Install `beacon-browser/SKILL.md` into the path for your host (table above) and start a **new** session. The skill is opt-in (`disable-model-invocation: true`) — invoke **`/beacon-browser`** or attach the skill to the prompt.

**MCP HTTP 401 / auth header rejected**  
Your bearer token is wrong or stale. [Setup](https://www.trybeacon.space/setup) → mint a new key → update **both** the MCP client config and the extension (side panel **Offline** / needs relink). Same `bcn_…` everywhere.

**“Can’t reach Chrome” but MCP shows connected**  
Connected only means HTTP to the relay. Run **`observe`** once; if that fails, fix extension link and token — don’t substitute a written shopping list for a real browser task.

**Silent clicks / nothing happens**  
Some sites block synthetic events. Use **`run` + `expect`**; clear **BLOCKED** overlays first; re-**observe** after SPA navigations (`Ctx spa` / ids may have changed).

**Wrong button / missed control**  
Use the **id from the latest observe**, not label guesswork. Stale id → **`NOT_FOUND`** → observe again. That accuracy model is what Beacon is for.

**Huge snapshots / slow loops**  
Observe is text, not pixels. Use **`observe({ id })`** or **`observe({ region })`** to zoom; prefer one **`run`** batch over dozens of serial tools.

**Not connected / timeout waiting for connection**  
Per-user MCP key; check tab roster on observe; reload the tab and extension after updates. Two connection timeouts in one chat → reconnect MCP host and retry.

**Login vs cloud browser**  
Your Chrome profile and cookies. The relay moves observe/act text you request — not a headless farm. [Privacy](https://www.trybeacon.space/privacy).

**Playwright MCP / sidepanel chat**  
This skill is for **Beacon MCP tools** in your agent host, not Playwright and not the Beacon sidepanel chat loop unless you explicitly want that.

**Anti-bot / heavy merchant walls**  
Built for **your** logged-in workflows (SaaS, grocery, internal tools), not bypassing every bot wall.

**Repo name**  
Default **`beacon-skill`**. Fork names: `beacon-browser-skill`, `beacon-mcp-skill`.

More: [trybeacon.space FAQ](https://www.trybeacon.space/#faq).

## License

Skill text is meant to be copied freely into personal skill paths. Beacon extension and service terms apply to the product itself.
