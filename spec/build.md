# Build

Step-by-step instructions for building Logos from the spec. Read `architecture.md` first — it defines the contracts this document tells you to implement.

All paths in this document are **relative to the workspace root** (the directory that contains `spec/`, `agent/`, `config/`, etc.). Run all commands from there.

The bootstrap reads from `spec/` (recipes, skills, cron defaults, this document) and generates code in `agent/`. After bootstrap, the running agent reads from `spec/` (skills, cron) and `agent/` (its own implementation), with `config/` for instance overrides.

## Before you start

Ask the user for:

- **Primary channel** (required) — Which messaging platform? (Telegram, WhatsApp, Discord, Slack, etc.)
- **AI model** (optional) — Which model to use. Defaults to the latest Claude Sonnet if not specified.

Don't worry about the assistant's name or personality — those are configured on first run through the messaging channel itself. The agent prompts the user and writes `config/SOUL.md` automatically.

## Key packages

- `ai` — **Vercel AI SDK**. Use the latest version. Provides `generateText` with built-in tool execution. Limit the number of tool-use steps to prevent runaway loops. Do not manually implement a tool loop.
- `@ai-sdk/anthropic` — Anthropic provider for the AI SDK (default). Can be swapped for `@ai-sdk/openai`, `@ai-sdk/google`, etc.
- `js-yaml` — YAML parsing for cron frontmatter and skill frontmatter
- `node-cron` — cron expression scheduling
- `tsx` — TypeScript execution without a build step. The agent can modify its own source and restart to apply changes.
- `zod` — schema validation, required by the AI SDK for tool parameter definitions
- `dotenv` — load environment variables from `config/.env`
- Channel-specific libraries — see the chosen recipe in `spec/channels/`

## Environment variables

All secrets (API keys, bot tokens) go in `config/.env`. This file is gitignored (the entire `config/` directory is gitignored by the spec repo). Load it at startup with `dotenv` pointed explicitly at `config/.env`.

At minimum:

- The API key for whichever AI provider you're using (e.g. `ANTHROPIC_API_KEY` for Anthropic)
- `AI_MODEL` — model to use. Default to a sensible current model; don't pin exact version strings since model names change frequently.
- `PRIMARY_CHANNEL` — the channel used for the owner's main conversation (e.g. `telegram`). The scheduler sends replies here.
- `LOGOS_SELF_EDIT` — `true` (default) or `false`. See `architecture.md` → Self-modification for the toggle's effects.
- `LOGOS_TERMINAL` — `true` (default) or `false`. When false, the terminal channel doesn't bind the socket and `agent/logos chat` has no daemon to connect to.
- `LOGOS_TERMINAL_SOCKET` — optional override for the terminal socket path. Defaults to `runtime/logos.sock`.
- `LOGOS_WEB_FETCH` — `true` (default) or `false`. When false, the `web_fetch` tool refuses every call.
- `LOGOS_WEB_FETCH_BACKEND` — `fetch` (default), `jina`, or `playwright`. See `spec/tools/web_fetch.md` for tradeoffs and privacy considerations.
- `LOGOS_WEB_FETCH_TIMEOUT_MS` — optional, default `15000`.
- `JINA_API_KEY` — optional; only consulted when backend is `jina`.

Channel-specific variables (including the owner's ID on that platform) are listed in each channel recipe under `spec/channels/`. Tool-specific variables are listed in each tool recipe under `spec/tools/`.

## Step-by-step

**Note:** Default cron jobs (`spec/cron/heartbeat.md`, `spec/cron/nap.md`, `spec/cron/dream.md`) and bundled skills (`spec/skills/`) ship with the spec. The running agent reads these directly. Don't copy them into `agent/`.

### 1. Initialize the project

- Create `agent/package.json` and `agent/tsconfig.json`
- Target ES2022 with Node module resolution
- Keep configuration minimal
- Install packages with `npm install <package-name>` from inside `agent/` rather than writing `package.json` by hand — this ensures you get the latest versions and only lists direct dependencies. **Never manually edit the `dependencies` or `devDependencies` objects in `package.json`.**
- Source code goes in `agent/src/` per the file structure in `architecture.md`.
- Use `process.cwd()` for the workspace root path, not `import.meta.dirname` — tsx runs in CJS mode where `import.meta.dirname` is undefined. The wrapper script ensures the process runs with the workspace root as cwd.
- Create `config/` if it doesn't exist (the agent should do this on first run, but the build can pre-create it). Create a `config/.env` template with the API key for the chosen provider, `AI_MODEL`, `PRIMARY_CHANNEL`, and any channel-specific variables. Leave secrets blank for the user to fill in.

### 1a. Initialize Git repos for agent/, config/, memory/

For each of `agent/`, `config/`, `memory/`: if the directory does not already contain a `.git/`, run `git init` and make an initial commit. This is **required for `agent/`** — the safe-restart auto-revert relies on `HEAD` existing. Without this, a self-edit that crashes on startup can't be rolled back. See `architecture.md` → Git repos per domain for the rationale.

Per-domain specifics:

- **`agent/`** — write `agent/.gitignore` (ignore `node_modules/`, `*.log`, `*.pid`). Then `git add -A && git commit -m "bootstrap"`.
- **`config/`** — write `config/.gitignore` that ignores `.env`, `.env.*` (and allows `.env.example` if used). **Never commit `.env`** — it contains secrets. After writing the gitignore and the `.env` template, `git add .gitignore && git commit -m "initial config"`. The `.env` file stays untracked.
- **`memory/`** — write `memory/.gitignore` (ignore `.obsidian/` for Obsidian vault settings). `git add -A && git commit -m "initial state"`. If `memory/` is empty, commit just the gitignore.

If any of these directories already has a `.git/`, leave it alone — the user is managing it.

**Commit additional changes as you make them.** If the bootstrap process edits generated files after the initial commit (e.g., you notice a bug and fix it), make a follow-up commit. End the bootstrap with a clean working tree in each repo.

### 2. Set up message storage

Implement `agent/src/threads.ts`. Storage format and semantics are defined in `architecture.md` → Storage / Message history.

Two functions to export:

- `appendMessage(channelId, conversationId, role, text, timestamp)` — ensure the parent directory exists, then append one JSON line with a trailing `\n`. Use `fs.promises.appendFile`. Sanitize `channelId` and `conversationId` to safe path segments — reject anything with `/`, `..`, or other path-escape characters.
- `getHistory(channelId, conversationId, limit)` — read the file (return empty if missing), split on newlines, parse each line as JSON, drop empty/malformed lines, return the last `limit` entries.

The router serializes per-conversation, so `appendMessage` never has a concurrent writer on the same file. No locking needed.

### 3. Build the router

Implement `agent/src/router.ts`. Behavior contract is in `architecture.md` → Router (especially the `NO_REPLY` semantics and the [Channel `send()` contract](architecture.md)).

Implementation:

- Per-conversation queue using a `Map<conversationId, Promise>` so each conversation runs serially.
- For each inbound message: append to JSONL via `threads.appendMessage`, fetch history via `threads.getHistory`, hand history to the agent, append the assistant reply to JSONL, call `channel.send(conversationId, reply)`.
- The reply is stored and sent **as-is** — including the literal string `NO_REPLY`. No translation. Channels detect `NO_REPLY` themselves per the contract.

### 4. Build the agent

Use the Vercel AI SDK's `generateText` for automatic tool execution. Limit the number of tool-use steps to prevent runaway loops. Do not manually implement a tool loop.

- Use the `@ai-sdk/anthropic` provider by default.
- Read the model from the `AI_MODEL` environment variable with a sensible default.
- The agent receives conversation history (the current message is already the last entry). Pass it directly to the SDK as the messages array.
- Cap conversation history at 50 messages (most recent) to avoid blowing past token limits. Apply the cap when retrieving history, not in the agent.
- Guard against oversized prompts. Estimate tokens using a 4:1 character-to-token ratio. First, truncate any individual message over 10,000 tokens. Then, if the total (system prompt + messages) exceeds 150,000 tokens, drop the oldest messages until it fits.

#### System prompt assembly

The system prompt is concatenated from these sections, in order:

1. **`config/SOUL.md`** (identity) — if missing, run the first-run flow (see step 4a).
2. **Memory manifest** — see `architecture.md` → Memory format → Loading into context. Build it from the memory module's manifest output (step 4b).
3. **Last 24 hours of `memory/journal/` entries inline** — recent agent-authored notes likely to be relevant; older journal entries appear in the manifest only.
4. **Skills summary** — names and descriptions from `spec/skills/*/SKILL.md` and `config/skills/*/SKILL.md` frontmatter; config wins on name collision. Skills loader described in step 4b.
5. **Today's date** — so `remember` and other date-aware behavior work without a tool call.

#### 4a. First-run flow

If `config/SOUL.md` doesn't exist when the agent assembles its system prompt:

- Use a minimal bootstrap system prompt: "You are a new personal AI assistant named Logos. You haven't been configured yet. On your next reply, introduce yourself and ask the user (1) what to call yourself and (2) how you should act — personality, tone, style. Once they answer, use `write_file` with `mode: \"create\"` to create `config/SOUL.md` with the chosen name and personality. Nothing else belongs in that file — memory, skills, and other context are loaded separately."
- After the user answers, the agent writes `config/SOUL.md`. Subsequent invocations read it normally.
- Also ensure `config/`, `memory/`, and `runtime/` directories exist; create them if not.

#### Tools

Implement one tool per recipe under `spec/tools/`. Each recipe is the contract for that tool — inputs, outputs, behavior, dependencies.

Bundled tools to build (each has a recipe):

- `read_file`, `write_file`, `edit_file`, `find_memory`, `remember`, `shell`, `delegate_task`, `web_fetch`

Implementation conventions:

- **Path-safety helper** — share a single `agent/src/tools/_paths.ts` (or similar) used by every path-taking tool. The helper resolves to absolute, rejects paths escaping the workspace root, and enforces the `spec/` and `agent/` write guards (see `architecture.md` → Self-modification for what those guards are and when they apply).
- **Memory graph cache invalidation** — any tool that writes under `memory/` (`write_file`, `edit_file`, `remember`) must delete `runtime/memory-graph.json` so the next `find_memory` rebuilds.
- **Tool return shapes** follow `architecture.md` → Tool return shapes. Never return bare `null` from a tool.

Custom tools are added by dropping `.ts` files directly into `agent/src/tools/` alongside the built-in ones. Loader scans that single directory.

#### 4b. Memory module

Implement `agent/src/memory.ts`. Resolution rules, manifest format, backlink semantics, and Obsidian compatibility are defined in `architecture.md` → Memory format.

The module exports:

- **Link resolver** — given a name (optionally path-qualified), return the matching file path or `null` per the resolution rules. Does NOT auto-create.
- **Graph builder** — at startup, scan `memory/**/*.md`, parse frontmatter and `[[...]]` links, build a name index and a backlink index covering the full tree. Build manifest entries for **root-level files only** (`memory/*.md`, not recursing into subfolders) — subfolder files are reachable via `[[wiki-links]]` and don't appear in the system prompt.
- **Cache** — write the graph to `runtime/memory-graph.json`. On startup, check the cache: if all source file mtimes are ≤ the cache's mtime, use it; otherwise rebuild.
- **Frontmatter parser** — use `js-yaml` to parse the YAML block between `---` delimiters. Tolerate missing or malformed frontmatter (treat as empty).

Skills loader: scan both `spec/skills/` and `config/skills/` for directories containing `SKILL.md`. Extract the YAML frontmatter with `js-yaml` (not regex). On name collision, `config/` wins.

#### 4c. Self-edit enforcement

Read `LOGOS_SELF_EDIT` once at startup and thread the boolean through the tool loader and skills loader. Behavior is defined in `architecture.md` → Self-modification.

What you must implement:

- **Always-on `spec/` write guard** in the path-safety helper: any path under `{workspace}/spec/` throws `spec/ is read-only at runtime; instance-specific changes belong in config/`. Independent of `LOGOS_SELF_EDIT`.
- **Conditional `agent/` write guard** when `LOGOS_SELF_EDIT=false`: paths under `{workspace}/agent/` throw `self-edit is disabled; refusing to write under agent/`.
- **Skills loader filter** when `LOGOS_SELF_EDIT=false`: skip the `self-edit` directory in `spec/skills/` so the skill is hidden from the agent's prompt.
- **Shell tool description nudges**: always include the `spec/` warning; conditionally append the `agent/` warning when `LOGOS_SELF_EDIT=false`. These are conventions, not enforcement — see Sandboxing below for hard enforcement.

#### 4d. Sub-agent runner

Implement `agent/src/agents/runner.ts`. Tool contract is in `spec/tools/delegate_task.md`; design rationale and constraints are in `architecture.md` → Sub-agents.

Single exported function `runSubAgent({ prompt, skills, tools, model })`:

- **Resolve skills** by name from `spec/skills/` then `config/skills/` (config wins). Read the FULL `SKILL.md` body. Return `{ ok: false, error }` if any name is missing.
- **Resolve tools** from the loaded tools map (the same map `agent.ts` builds). **Always strip `delegate_task`** — sub-agents never recurse, regardless of caller request.
- **Build the system prompt:** framing line + today's date + concatenated skill bodies (NO SOUL.md, NO memory manifest, NO conversation history).
- **Call `generateText`** with the system prompt, the `prompt` as the user message, the resolved tool subset, the model from the argument or `AI_MODEL`, and a step limit equal to the main agent's (e.g. `stepCountIs(25)`).
- **Log the invocation** to `runtime/logs/` (or a sub-agents-specific logfile) with timestamp, skills, tools, model, duration, token usage. Do NOT include the prompt or response text — could contain sensitive content.
- **Return** `{ ok: true, response: text }` or `{ ok: false, error }`.

Framing line for the system prompt:

> "You are a focused sub-agent invoked by the main Logos agent. Complete the task described in the user message and return your final response. You have no access to the main agent's identity, memory, or conversation history beyond what's stated below."

### 5. Build the channel registry

Implement `agent/src/channels/index.ts`. Channels are defined in `architecture.md` → Channels (especially the [`send()` contract](architecture.md)).

- Scan `agent/src/channels/` for `*.ts` files at startup. For each, dynamically import and call its `register()` function with the router. No manual registration list.
- A channel's `register()` returns `{ channelId, ownerConversationId, send }` if it connected, or nothing if credentials were missing. When skipping, log which environment variables are missing and how to get them (see the recipe in `spec/channels/{name}.md`).
- The registry collects connected channels into a map by channel ID so the scheduler can look up any channel's send function and owner conversation ID.
- If no channels connected, exit with a clear error.

### 6. Build the user's chosen channel

Read the recipe at `spec/channels/{name}.md` and follow its setup instructions. The implementation goes at `agent/src/channels/{name}.ts`. The channel must only forward messages from the owner (identified by the owner ID in the recipe's environment variables) and silently ignore everyone else. Get the full loop working end-to-end before adding anything else.

### 6b. Build the terminal channel (always included)

Terminal is a zero-dependency, bundled channel shipped with every bootstrap. Follow `spec/channels/terminal.md` for the protocol, replay semantics, and rendering rules. Implementation lives at `agent/src/channels/terminal.ts` (server) and `agent/src/cli/chat.ts` (client). Wired into the wrapper as `agent/logos chat` (see step 9).

The `LOGOS_TERMINAL=false` env var disables the channel: the server doesn't bind the socket, and the channel registry skips it.

### 7. Build the scheduler

Implement `agent/src/scheduler.ts`. Cron format, merge rules, and the merged-job table are defined in `architecture.md` → Scheduler.

- Scan both `spec/cron/` and `config/cron/`, parse frontmatter with `js-yaml`, merge per the layering rules (frontmatter override, body append; skip when merged `enabled: false`).
- Use `node-cron` to schedule each enabled job using its merged `schedule` field.
- When a job fires, append a reminder to the merged body: "If you have nothing to say to the owner, respond with NO_REPLY." Then look up the primary channel and dispatch through the router as a synthetic message addressed to the owner's conversation.
- **Honor the `history:` frontmatter field** (default `primary`). For `history: none`, dispatch the synthetic message with a flag that tells the router to skip `getHistory` and run the agent with only the synthetic prompt as input. Add a corresponding option to the router's dispatch path.
- Add a CLI subcommand `agent/logos cron` that prints the merged job table with source annotations (`[spec]`, `[config]`, `[spec → overridden by config]`, `[disabled]`).

### 8. Wire it all together

The entry point (`agent/src/index.ts`):

1. Loads `config/.env` via dotenv
2. Ensures `runtime/` exists
3. Registers channels
4. Starts the scheduler
5. Logs that it's running

That's it. No HTTP server needed unless a channel requires a webhook.

### 9. Create the `agent/logos` wrapper script

Create a bash script at `agent/logos` that supports:

- `agent/logos` or `agent/logos start` — start the process in the background
- `agent/logos stop` — stop it
- `agent/logos restart` — restart it (with safe-restart protocol below)
- `agent/logos status` — check if it's running
- `agent/logos cron` — show the merged cron job table
- `agent/logos chat [flags]` — connect a terminal client to the running daemon (see step 6b). Passes flags through to `npx tsx agent/src/cli/chat.ts`. Fails fast with "daemon not running; start it with `agent/logos start`" if the daemon PID isn't alive.

The script must be invoked from the workspace root (the parent of `agent/`). It should `cd` to the workspace root if invoked from elsewhere by resolving its own location.

Run the process with `npx tsx agent/src/index.ts` (not compiled JS). This way the agent can modify its own TypeScript source and restart to apply changes — no build step needed.

Use a PID file at `runtime/logos.pid` and write logs to `runtime/logs/`. Both are gitignored. Append to the log file — don't truncate it on restart.

**`stop` must kill the entire process tree, not just the PID file's PID.** `npx tsx agent/src/index.ts` produces a multi-process tree (npm wrapper + tsx node child). If `stop` only kills the npm wrapper, the node child gets re-parented to init and survives — invisible to the wrapper, still polling channels, still firing scheduled jobs. Each `restart` then leaks a zombie daemon.

Walk the tree with `pgrep -P` and kill recursively:

```bash
kill_tree() {
  local pid=$1
  local children
  children=$(pgrep -P "$pid" 2>/dev/null || true)
  for child in $children; do
    kill_tree "$child"
  done
  kill -TERM "$pid" 2>/dev/null || true
}
```

`pgrep` ships on both macOS and Linux. After SIGTERM, give the process a few seconds to shut down gracefully, then SIGKILL the root PID if it's still alive.

**Safe-restart protocol for `restart`** (architecture: see `architecture.md` → Self-modification):

1. **Snapshot the current `agent/` HEAD.** If `agent/` is a Git repo (`agent/.git` exists), record the current commit SHA with `git -C agent rev-parse HEAD`. If it's not a Git repo, skip the rollback steps below and warn the user that self-edit rollback won't work.
2. **Typecheck.** Run `tsc --noEmit -p agent/tsconfig.json`. If it fails, abort the restart, keep the old process running, and print the error.
3. **Stop the old process** (if running) and **start the new one** in the background.
4. **Health check.** Wait a few seconds (e.g. 5s) and check if the new PID is still alive. If dead:
   - Print the last ~30 lines of the log so the user can see what crashed.
   - If `agent/` is a Git repo and HEAD has moved since the snapshot: run `git -C agent reset --hard <snapshot-sha>` to revert the self-edit, then start the process again with the pre-edit code. Log the auto-revert clearly (`[logos] post-start check failed; auto-reverted agent to <sha> and restarted`).
   - If no Git repo or HEAD is unchanged: leave the process stopped and tell the user to investigate.

Auto-revert only affects commits the agent itself made between the snapshot and the failed restart. Changes in `config/` or `memory/` are not part of self-edit.

## Before you're done

Verify the build before handing it off. Run these checks outside any sandbox so that tools like `tsx` work normally:

- `tsc --noEmit` (against `agent/tsconfig.json`) passes with no errors
- `agent/logos start` with blank credentials fails and shows the error in the terminal
- `agent/logos start` with blank credentials then `agent/logos status` reports not running
- `config/SOUL.md` does not exist yet (it's written on first run, not by the build)
- `spec/cron/heartbeat.md`, `spec/cron/nap.md`, and `spec/cron/dream.md` are unchanged
- `spec/` is untouched by the build (the bootstrap only writes to `agent/` and optionally creates `config/`)
- The wrapper script is executable (`chmod +x agent/logos`)
- `git -C agent log --oneline` shows at least one commit (the `bootstrap` anchor — critical for safe-restart auto-revert)
- `git -C agent status` is clean (no uncommitted changes)
- `git -C config log --oneline` shows at least one commit; `config/.env` is NOT in the commit
- `git -C memory log --oneline` shows at least one commit

## Sandboxing (optional, for hard self-edit enforcement)

`LOGOS_SELF_EDIT=false` (step 4c) uses tool-level guards — soft enforcement. A determined agent with `shell` access can bypass them. For guaranteed enforcement, run the process in a sandbox with `agent/` and `spec/` mounted read-only. Pick the approach that fits your OS:

**Linux (read-only bind mount):**

```bash
sudo mount --bind agent/ agent/
sudo mount -o remount,ro,bind agent/
```

Or use `bwrap` / `firejail` to namespace-sandbox the process with a read-only view of `agent/` and `spec/`.

**macOS (`sandbox-exec`):**

```bash
# sandbox.sb denies all writes under agent/src/
(version 1)
(allow default)
(deny file-write*
  (subpath "/absolute/path/to/workspace/agent/src"))

sandbox-exec -f sandbox.sb agent/logos start
```

**Either platform (separate user):**

Run the agent as a user that only has read access to `agent/` and `spec/` on the filesystem. Owner owns the directories; agent user can read + execute but not write.

None of these are wired into the stock build — they're deployment choices for users who need hard enforcement.

## When you're done

Remove the reference to this file from `AGENTS.md` and `CLAUDE.md`. The build is complete.
