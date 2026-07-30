---
name: browser-use
description: Drive a real browser via browser-harness when webFetch isn't enough — auth, clicks, forms, screenshots, signed-in pages.
preferred_model: reasoning
---

# Browser use

Three ways to read the web, in increasing cost and blast radius. Pick the cheap one first.

1. **`webFetch` tool** — anonymous reads of public-ish pages. Articles, GitHub HTML, docs, blog posts. No setup. **Try this first.** Behavior varies by backend (Claude/Codex use native hosted fetch with JS rendering; Vercel/`openai` use agent-sdk's bundled in-process impl; future plans swap that for Tavily Extract — see `plan/web-search-and-fetch.md`).
2. **`browserFetch` tool** — single-shot headless-Chromium render with Mozilla Readability cleanup. Still anonymous (no shared session with the owner's Chrome), but executes JS and strips chrome down to the article body. Reach for it when `webFetch` returns junk because the page is JS-heavy, bot-blocks plain HTTP, or hides the article behind a paywall preview a logged-out browser still gets. See `spec/tools/browser_fetch.md`.
3. **`browser-harness` CLI via `bash`** — a real, attached Chrome session with whatever sessions are signed in. Use only when neither anonymous tier works: signed-in pages, multi-step forms, clicks, file uploads, screenshots, anything CDP-only.

Escalate left-to-right when the previous tier returns junk, an empty body, a login wall, or a captcha. Don't jump straight to `browser-harness` — `browserFetch` clears most JS-rendering and bot-blocking cases without spending the owner's logged-in session.

## Privacy

By default `browser-harness` drives a **dedicated agent Chrome profile** at `~/.browserface/chrome` — separate from the owner's daily-driver Chrome. Cookies, history, tabs, and extensions are fully isolated. You can only see sites the owner has explicitly signed into the agent profile (typically Gmail, Slack, calendars — whatever the agent needs). The owner's banking, work email, and personal tabs in their own Chrome are unreachable, even if you tried.

Treat what the agent profile *does* contain as sensitive — it's still real session cookies for whatever services live there. Don't paste page contents into third-party services, don't log full screenshots, and prefer `delegate_task` for multi-step workflows so the raw context stays out of the main thread.

If the owner explicitly opts into the daily-driver Chrome instead (see "Fallback" below), the trust model expands dramatically — anything in any tab is fair game. The cautions above apply with extra weight.

Remote handoff via `browser/share` exposes the bridge over the public internet. The auth flag is mandatory unless the owner explicitly waives it (see Sharing setup), regardless of which Chrome you're driving.

## One-time install

### browser-harness

```bash
which browser-harness
```

If missing, walk the owner through installation. Confirm before you start:

```bash
git clone https://github.com/browser-use/browser-harness vendor/browser-harness
uv tool install -e ./vendor/browser-harness
```

If `uv` isn't installed, point the owner at https://docs.astral.sh/uv/ — don't try to install it for them, package managers are the owner's call.

### browserface

```bash
ls vendor/browserface/browser/start 2>/dev/null
```

If missing:

```bash
git clone https://github.com/browserface/browserface vendor/browserface
```

No global install — invoke as `vendor/browserface/browser/start` (or `…/browser/face`, `…/browser/share`) directly. Scripts bootstrap their own deps on first run.

### Sign into the agent profile

After both are installed, bring up the agent Chrome and ask the owner to sign into the sites you'll need:

```bash
vendor/browserface/browser/start
```

This launches a Chrome window using the dedicated agent profile (`~/.browserface/chrome`). Tell the owner:

> I've opened a Chrome window using the agent's dedicated profile. Please sign into [Gmail, Slack, …] in those tabs, then close the window when you're done. Cookies persist in this profile, so you only need to sign in once per site.

Subsequent agent invocations reuse those sessions automatically. The owner's daily-driver Chrome is never touched.

## Usage

Every invocation prefixes with the env-var bringup so `browser-harness` attaches to the dedicated agent profile rather than auto-discovering the owner's daily-driver Chrome:

```bash
export BU_CDP_WS=$(vendor/browserface/browser/start)
browser-harness <<'PY'
new_tab("https://news.ycombinator.com")
wait_for_load()
print(page_info())
PY
```

`browser/start` is idempotent — fast no-op when the agent Chrome is already running. The `BU_CDP_WS` env var tells `browser-harness` which CDP endpoint to attach to (overriding its default profile-dir discovery, which would otherwise find the daily-driver Chrome and re-trigger the per-connect Allow popup on every invocation).

Helpers (`new_tab`, `wait_for_load`, `page_info`, `screenshot`, `click`, `type_text`, …) are preloaded — call them directly, don't import. The full helper surface lives in `vendor/browser-harness/helpers.py`.

If `browser-harness --doctor` reports it's attached to the wrong Chrome (the daemon got stuck on the daily-driver from a previous session, e.g. before the agent profile existed), kill the daemon so the next call re-bootstraps with the current `BU_CDP_WS`:

```bash
pkill -f browser-harness
```

### Picking a click primitive

For anything addressable in the DOM (buttons, links, "Show more" expanders), prefer `js()` calling `element.click()` (or `dispatchEvent(new MouseEvent('click', { bubbles: true }))`) over the coordinate-based `click(x, y)`. Coordinate clicks against `getBoundingClientRect()` look reliable but break on any reflow between extraction and click — the page can shift, a layout pass can move the target, and you fire into empty space. JS-targeted clicks bind to the element itself, survive reflow, and don't require the element to be in the viewport.

`click(x, y)` stays the right tool for: canvas widgets, sites with custom event handlers that explicitly check synthetic-event shape (some banks, some games), and any pixel target that has no DOM handle.

### Heavy or multi-step work → delegate

Page contents and screenshots blow up context fast. For research, scraping, or any multi-step browser task, use `delegate_task` with `tools: ["bash"]` and a skill list that includes `browser-use`. The raw output stays in the sub-agent's context; only the summary comes back.

## Handing off to the owner

When the task hits something only a human can clear — login, 2FA, captcha, a confirmation you shouldn't auto-click — don't loop. Hand off via **browserface**, which gives the owner the same primitives (click, type, scroll, navigate) over a screenshot stream against the same agent Chrome you've been driving.

Two modes:

- **Local** (`vendor/browserface/browser/face`) — owner is at the same machine and opens `http://127.0.0.1:8768`.
- **Remote** (`vendor/browserface/browser/share`) — fronts the bridge with an ngrok tunnel so the owner can take over from elsewhere (phone, another machine).

`browser/face` defaults to the same agent profile, so the handoff is seamless: owner clears the wall in their tab, you resume on the same session.

### Sharing setup

`browser/share` requires two pieces of one-time owner setup.

**1. ngrok authtoken.** Check:

```bash
ngrok config check
```

If it errors, walk the owner through `ngrok config add-authtoken <token>` using their token from https://dashboard.ngrok.com/get-started/your-authtoken. The token is the owner's ngrok account; don't try to mint or substitute one.

**2. Default auth policy.** Check what browserface has saved:

```bash
vendor/browserface/browser/config show auth
```

If it prints `(unset)`, ask the owner what their auth policy should be and save it once:

```bash
vendor/browserface/browser/config set auth --oauth google --oauth-allow-email you@example.com
```

A remote share exposes the agent Chrome over the public internet, so `(unset)` means you stop and ask, not assume.

Save to memory only the **stable domain** if they have one (e.g. `you.ngrok.app`). After setup, every share is:

```bash
vendor/browserface/browser/share you.ngrok.app
```

(drop the domain for a fresh assigned URL). `browser/share` reads the saved policy from `~/.browserface/auth` automatically.

### When to launch

- **You hit a wall** — tell the owner what you need and offer to start a session; don't keep retrying.
- **Owner asks to watch or pair** — start one.

Don't launch unprompted just in case. If `browser/config show auth` returns `(unset)`, walk the owner through the setup above before running.

## Fallback: drive the owner's daily-driver Chrome

When the owner explicitly wants the agent to use their everyday Chrome (with all their existing logged-in sessions — Gmail, work, banking — and matching extensions/themes), there are two opt-in paths.

**`browser-harness`:** unset `BU_CDP_WS` so it falls back to its default discovery, which probes well-known profile dirs:

```bash
unset BU_CDP_WS
browser-harness <<'PY'
...
PY
```

The owner has to enable Chrome's per-profile remote-debugging toggle once: tick **"Allow remote debugging for this browser instance"** at `chrome://inspect/#remote-debugging`. The page shows "Server running at: 127.0.0.1:9222" once it's live. Whether they leave it on or toggle it per-task depends on how comfortable they are leaving the door open while not using the agent.

**`browser/face`:** pass `--discover`:

```bash
vendor/browserface/browser/face --discover
```

Both paths trigger Chrome's **per-connect Allow popup** the first time each new browser session attaches. The symptom is a non-zero exit with stderr containing **`click Allow on chrome://inspect (and tick the checkbox if shown)`**. Treat that exact stderr as a wait-for-human signal: message the owner once asking them to click Allow in their Chrome window, then wait for their reply. Don't loop.

**Privacy:** in this mode you're driving the owner's full browsing — banking, work email, personal tabs, all of it. Apply the cautions in the Privacy section with extra weight, and prefer `delegate_task` more aggressively to keep raw page contents out of the main thread.

## Extending helpers.py

If a needed helper is missing (the upstream design assumes this happens often — that's the point of browser-harness), follow the **`coding` skill** with the working directory set to `vendor/browser-harness/` and a prompt like:

> Add a function to `helpers.py` that does X. Keep it small and consistent with the existing helpers. Commit on a local `protos-local` branch so upstream pulls rebase cleanly.

Then re-run your task. Because browser-harness was installed with `uv tool install -e`, the new helper is live immediately — no reinstall.

Don't hand-edit `helpers.py` from your `bash` tool. Delegate to the coding agent so the change is reviewed, committed, and cleanly mergeable against upstream.

## Site-specific patterns

`vendor/browser-harness/domain-skills/` contains community-contributed patterns for specific sites (github, linkedin, amazon, …). **First action on a new domain:** `ls vendor/browser-harness/domain-skills/{site}/` and skim anything that's there. Even a 30-second read saves selector hunting and avoids fragile improvisations against site UIs that change often (X's "Show more" expansion is a classic example).

When you discover a non-obvious pattern yourself (a private API, a stable selector, a needed wait, a reliable way to dismiss a recurring dialog), file it under `vendor/browser-harness/domain-skills/{site}/` via the `coding` skill at the end of the task. Per upstream convention, don't hand-author these — let the coding agent generate them from what actually worked in the browser this run.

## When to refuse

- The owner asked for something that would require their auth on a service they haven't used in this thread or in memory. Confirm first — "I'd be using your logged-in {service} session for this; OK?"
- Any action that spends money, sends a message, or makes a public post. Confirm.
- Anything resembling scraping at scale, automated account creation, or evading rate limits / bot detection. Refuse and explain.
