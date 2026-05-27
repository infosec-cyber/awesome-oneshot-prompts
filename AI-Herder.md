# One-shot prompt: build `harness` (herder + supervisor) from scratch

> Paste everything below the line into a fresh Claude Code session in an empty
> directory. It is self-contained — no reference to the original repo needed.

---

Build me a complete multi-agent orchestration harness for running fleets of
Claude Code CLI sessions inside isolated tmux servers, watched by a web UI and
auto-nudged by an LLM supervisor when they go idle. Produce **every file**
listed under "Deliverables" exactly at those paths, runnable with no edits.

## What it is

Two long-running daemons sharing one SQLite db:

1. **herder** (Node) — web grid of live xterm.js tiles, each attached over
   websocket → `node-pty` → `tmux -L agent-<name> attach`. REST API to
   spawn/kill/meta agents, SSE stream of activity + open questions, embedded
   VS Code web reverse-proxy, command palette / search / matrix-mode frontend.
2. **supervisor** (Python, asyncio) — 30 s tick loop. For each agent: capture
   pane, classify `running|idle|waiting_operator|crashed`, detect permission
   dialogs via regex YAML, enqueue "questions" for the operator, and when an
   `auto=1` agent idles past threshold, shell out to `claude -p <persona>` to
   generate a one-line nudge in the operator's voice and `tmux send-keys` it.

Shared state lives in `/var/lib/harness/harness.db` (WAL). Agents are tmux
sessions named `main` on socket `agent-<name>` so a SIGSEGV in one tmux server
never cascades. Spawning goes through `bin/agent-spawn` which builds the
`claude --dangerously-skip-permissions [--model M] [--resume UUID]` command,
deterministically derives `--session-id` from the agent name, pre-trusts the
cwd in `~/.claude.json`, and starts tmux with `harness/tmux.conf`.

## Deliverables (create all of these)

```
harness/
├── schema.sql
├── tmux.conf
├── detectors.yml
├── bin/
│   ├── agent-spawn        (bash, +x)
│   ├── agent-send         (bash, +x)
│   ├── agent-kill         (bash, +x)
│   ├── agent-restart      (bash, +x)
│   ├── restart-on-idle    (bash, +x)
│   ├── harnessctl         (python3, +x)
│   └── harness-migrate    (python3, +x)
├── personas/
│   ├── hacking.txt
│   ├── development.txt
│   ├── question.txt
│   └── retro.txt
├── host-config/
│   └── localhost.env      (template; systemd EnvironmentFile=host-config/%l.env)
├── systemd/
│   ├── harness.target
│   ├── harness-herder.service
│   └── harness-supervisor.service
├── supervisor/
│   ├── supervisor.py
│   ├── db.py
│   ├── tmux.py
│   ├── classify.py
│   ├── detectors.py
│   └── analyze.py
└── herder/
    ├── package.json
    ├── db.mjs
    ├── server.mjs
    └── public/index.html
```

## Hard interface contracts (do not deviate)

### Env vars
`HARNESS_ROOT` (default `/shared/harness`), `HARNESS_DB`
(`/var/lib/harness/harness.db`), `HARNESS_DEFAULT_KIND` (`hacking`),
`HARNESS_SUPERVISOR_MODEL`, `HARNESS_INSTRUCTIONS`, `HARNESS_MCP_CONFIG`,
`PORT` (`3077`), `TMUX_BIN`, `TARGETS_DIR`, `WORKSPACES_DIR`, `TRIAGE_URL`,
`TRIAGE_API_KEY`, `TRIAGE_KEYS_FILE`, `TRIAGE_K8S_NS`, `TRIAGE_MINT_DEPLOY`,
`CODE_SERVER_BIN`, `CODE_SERVER_HOST`, `CODE_SERVER_PORT_RANGE`,
`CODE_SERVER_BASE_PATH`, `CODE_SERVER_DATA_DIR`.

### tmux convention
Socket = `agent-<name>`, single session `main`, target string `=main:`. Helper:
`tmux -L agent-<name> <verb> -t =main: …`. `agent-kill` = `kill-server` on that
socket only.

### SQLite schema (`schema.sql`, idempotent — `CREATE TABLE IF NOT EXISTS`)
Tables, columns, CHECK constraints and seeded config rows **exactly**:

- **agents**(name PK CHECK glob `[A-Za-z0-9._-]*` len 1-64, cwd NOT NULL, cmd
  DEFAULT '', model, resume_id, kind CHECK IN
  ('hacking','development','question') DEFAULT 'hacking', grp, auto INT
  DEFAULT 1, state CHECK IN ('spawning','running','idle','waiting_operator',
  'parked','analyzing','crashed','dead','paused') DEFAULT 'spawning',
  state_since DEFAULT unixepoch(), idle_ticks INT DEFAULT 0, nudge_count INT
  DEFAULT 0, wake_at INT DEFAULT 0, last_output_sha, last_output_at,
  last_user_msg, pane_title, consec_crashes INT DEFAULT 0, q_timeout_s INT
  DEFAULT 900, q_default CHECK IN ('approve_safe','deny','park','skip')
  DEFAULT 'approve_safe', created_at, updated_at). Index on (state).
- **questions**(id PK AI, agent FK CASCADE, kind CHECK IN
  ('permission','choice','clarification','confirm','unknown'), detected_by,
  prompt, options JSON, suggested, status CHECK IN
  ('open','answered','defaulted','expired','superseded') DEFAULT 'open',
  answer, answered_by, created_at, deadline_at NOT NULL, closed_at). Partial
  index on (status,deadline_at) WHERE status='open'; index (agent,created_at
  DESC).
- **events**(id PK AI, ts DEFAULT unixepoch(), agent, actor NOT NULL, type NOT
  NULL, from_state, to_state, detail JSON). Index (agent,ts DESC), (ts DESC).
- **nudges**(id PK AI, agent FK CASCADE, ts DEFAULT unixepoch(), kind, msg,
  delivered INT DEFAULT 0). Index (agent,ts DESC).
- **config**(key PK, value). `INSERT OR IGNORE` seeds: tick_s=30,
  idle_nudge_after_ticks=4, default_agent_model='', supervisor_model='',
  model_profiles='', analyze_timeout_s=120, max_concurrent_analyze=4,
  stall_no_output_s=300, state_watchdog_s=1200, spawn_timeout_s=90,
  crash_backoff_max=3.

PRAGMAs at top: `journal_mode=WAL`, `busy_timeout=5000`, `synchronous=NORMAL`,
`foreign_keys=ON`.

### `tmux.conf`
`exit-empty off`, `exit-unattached off`, `default-command "bash -l"`,
`history-limit 50000`, `status off`, `escape-time 0`,
`default-terminal tmux-256color`, `window-size latest`,
`aggressive-resize on`, `focus-events on`, `prefix None`.

### `detectors.yml` (ordered, first-match-wins, hot-reloaded on mtime)
Seven entries with ids `rx:perm_numbered`, `rx:perm_phrase`,
`rx:trust_folder`, `rx:choice`, `rx:confirm_yn`, `rx:confirm_enter`,
`rx:clarify`. Each may have: `pattern` (required), `require`, `options_rx`
(captures key=group1 label=group2 from numbered menu lines like
`❯ 1. Yes …`), static `options`, `suggest`, `suggest_if_opt1` (regex applied
to first parsed option's label → if matches, suggest its key). Kinds map to
the questions.kind enum.

---

## `bin/` scripts

**agent-spawn** `<name> <cwd> [--model M] [--resume UUID] [--fork UUID] [-- CMD…]`
- Validate name regex; cwd must exist.
- If no custom CMD: build `claude --dangerously-skip-permissions`, append
  `--mcp-config $HARNESS_MCP_CONFIG` if file exists, `--model $MODEL` if set,
  `--append-system-prompt "$(cat $HARNESS_INSTRUCTIONS)"` if file exists.
  Compute `SID=$(uuidgen --sha1 -n @oid -N "harness-agent-$NAME")`. Choose
  `--resume $RESUME` | `--resume $FORK --fork-session --session-id $SID` |
  `--resume $SID` (if `~/.claude/projects/*/$SID.jsonl` exists) |
  `--session-id $SID`.
- `jq` patch `~/.claude.json` → `.projects[$CWD].hasTrustDialogAccepted=true`.
- If `tmux -L agent-$NAME has-session -t =main` already → print "exists", exit 0.
- `tmux -L $SOCK -f $ROOT/tmux.conf new-session -d -s main -x 220 -y 50 -c $CWD`;
  sleep 0.6; `send-keys -l "$CMD"`; sleep 0.2; `send-keys Enter`.

**agent-send** `<name> <text…>` — if text is literally `Enter`/`Escape`/`C-x`
send as key; else `send-keys -l "$TEXT"`, sleep 0.3, `send-keys Enter`. On
failure: send `C-c`, sleep 0.5, retry once. All `send-keys` wrapped in
`timeout 5`.

**agent-kill** `<name>` — `tmux -L agent-$NAME kill-server 2>/dev/null || true`.

**agent-restart** `<name>` — read cwd/model/resume_id from db, `agent-kill`,
`agent-spawn`, then `UPDATE agents SET state='spawning',state_since=unixepoch(),
consec_crashes=0,idle_ticks=0,nudge_count=0`.

**restart-on-idle** `<name> [max_s=3600]` — poll db every 15 s; when
`state='idle'` → `agent-restart`; abort on dead/paused.

**harnessctl** (python) — subcommands `ls`, `add`, `adopt`, `release`, `set`,
`pause|resume|park|kill`, `nudge`, `questions`, `answer`, `events`. All write
an `events` row with actor='harnessctl'. `ls` prints columns
NAME/STATE@HH:MM/KIND/AUTO/NC/Q?/GROUP. `answer <id> <text>` →
agent-send + close question + state=running.

**harness-migrate** (python) — import v1 state from
`~/.claude-supervisor/sessions.json`, `~/bin/tmux-fleet-restore` ROWS array
(`"name|dir|uuid"` lines), per-pane `name_*.json`, and live `tmux ls`. Merge,
infer kind from cwd hints, INSERT OR IGNORE into agents (state='paused'
auto=0), copy nudge history. `--dry-run` prints table only.

---

## `supervisor/` (python3, stdlib + pyyaml + optional `systemd.daemon`)

**db.py** — `DB` class wrapping a single autocommit `sqlite3.Connection`
(row_factory=Row, busy_timeout=5000, WAL, FKs). Methods: `config()`,
`agents()`, `agent(name)`, `update(name,**cols)`, `set_state`/`transition`
(writes events row actor='supervisor'), `event()`, `open_question(agent)`,
`enqueue_question(agent,kind,detected_by,prompt,options,suggested,timeout_s)`,
`close_question(id,status,answer=None,by=None)`, `expired_questions()` (join
agents for q_default), `add_nudge`, `recent_nudges(agent,n)`.

**tmux.py** — thin wrappers shelling to tmux/bin scripts: `alive`,
`capture(name,lines)`, `title`, `send`, `spawn`, `kill`.

**classify.py** —
`BUSY_RE = r'[·✶✳✢*∗]\s+\w+…|\besc to interrupt\b|\(\d+[ms].*tokens\)'`;
`is_working(tail)` = BUSY_RE in last 15 lines; `is_idle(tail)` = last 6
non-empty lines contain a bare `❯`/`>`/`›` or `› foo {`; `output_sha(tail,n)`
= sha1 of last n non-empty lines, first 16 hex.

**detectors.py** — `load()` parses detectors.yml (cached on mtime, compile
patterns I|M). `detect(tail)` runs ordered list against last 25 non-empty
lines; returns `(id, kind, options, suggested)` or None.

**analyze.py** — `analyze(name, content, kind, history, timeout, model,
agent_model)`: read `personas/<kind>.txt` (fallback hacking), build prompt =
persona + optional `This pane is running model <m>.` + previous-nudges block
+ `--- pane <name> ---\n<last 6000 chars>\n--- end ---`. Exec
`claude [--model M] --dangerously-skip-permissions -p <prompt>`, await with
timeout. On timeout/non-zero/empty → return `FALLBACK[kind]`. Else return the
**last non-empty line** of stdout. Export `PARK_RE = r'__PARK\s+(\d+(?:\.\d+)?)h__'`
and `FALLBACK` dict (hacking/development/question one-liners).

**supervisor.py** — `class Supervisor`:
- `reload_cfg()` reads config table → ints; builds `model_profiles` dict
  (default `{"":{idle_after:4,backoff_cap:8}, "opus":{2,4}, "sonnet":{2,3},
  "haiku":{1,2}}` if config empty); creates `asyncio.Semaphore(max_analyze)`.
- `run()` loop: `sd_notify READY=1`; each tick `sd_notify WATCHDOG=1`,
  `reload_cfg`, `apply_q_defaults`, `gather(tick_agent(a) for a in agents)`
  with `return_exceptions=True` (log + emit `tick_fail` event on error),
  emit `tick_overrun` event if tick took >2×tick_s, then
  `wait_for(stop, timeout=tick_s−elapsed)`.
- `tick_agent(a)`:
  - skip if name in `{"shortcake","admin"}` or state in (paused,dead).
  - parked: if `wake_at<=now` → transition→idle. return.
  - `if not tmux.alive` → `handle_crash`.
  - capture 60 lines; compute sha; if changed update
    `last_output_sha,last_output_at,pane_title`.
  - watchdog: if `now−last_output_at > watchdog_s` and state non-terminal →
    force re-classify (event `stuck`).
  - spawning: if classify busy/idle → running (`consec_crashes=0`); elif
    `> spawn_timeout` → if tmux alive transition→running
    (`spawn_timeout_alive`), else →crashed + handle_crash. return.
  - waiting_operator → `tick_waiting` (if no open q → running; if pane sha
    changed AND prompt-sha no longer matches tail → close q `superseded`,
    →running). return.
  - `detectors.detect(tail)` hit → `enqueue_question` (dedupe on prompt sha
    vs existing open q; supersede old one), transition→waiting_operator.
  - `is_working` → transition→running (reset idle_ticks,nudge_count) if not
    already.
  - `is_idle` → transition→idle on first; else `idle_ticks++`; pick
    `model_profile(a.model)`; threshold =
    `max(min(max(2, 2**nudge_count), backoff_cap), idle_after)`; if
    `auto && idle_ticks>=threshold` → `asyncio.create_task(nudge(...))`.
  - else (no classification) and `now−last_output_at > stall_s` and last
    line endswith `?:❯›>` → enqueue question `(stall:prompt_like, unknown)`.
- `nudge(name,kind,tail,agent_model)`: under semaphore,
  transition→analyzing; `history=recent_nudges(3)`; call analyze; if result
  matches PARK_RE → transition→parked with `wake_at=now+h*3600`; else
  `tmux.send`; `add_nudge`; on send-fail event + →crashed; on success
  →running with `nudge_count++`, `idle_ticks=0`.
- `handle_crash(a)`: `consec_crashes++`; event `crash`; if `>=crash_max`
  →dead; else `tmux.kill`+`tmux.spawn(cwd,model,resume,cmd)` →spawning or
  →dead on spawn fail.
- `apply_q_defaults()`: for each `expired_questions()` apply per-agent
  `q_default`: `approve_safe`+permission+suggested → send suggested,
  status=defaulted, →running; `deny` → find option whose label startswith
  No/Cancel else `Escape`, send, →idle; `skip` → send Escape, set auto=0,
  →idle; else (`park` / approve_safe with no sugg) → close defaulted, →parked
  wake_at=+1h.
- `main()`: install SIGTERM/SIGINT → `stop.set()`; run loop.

---

## `herder/` (Node 20+, ESM, deps: express ^4, ws ^8, node-pty ^1,
better-sqlite3 ^11; CDN: @xterm/xterm 5.5.0 + addon-fit 0.10.0)

### `db.mjs`
Open `better-sqlite3` on `HARNESS_DB`, WAL, busy_timeout=5000, FKs on.
Prepared statements + exports: `config()`, `agents()`, `agent(name)`,
`insertAgent({name,cwd,cmd,model,resume_id,kind,grp,auto,state})` (+create
event), `update(name,cols)` (adds updated_at), `transition(name,from,to,
trigger,extra)` (update + event), `event(agent,type,{from_state,to_state,
detail})` actor='herder', `openQuestions()` (join agents for kind/grp),
`question(id)`, `answerQuestion(id,ans,by)`, `closeQuestion(id,status,by)`,
`events(agent,limit)`, `deleteAgent(name)`, `now()`, `DB_PATH`.

### `server.mjs`

Constants: `VALID_NAME=/^[A-Za-z0-9._-]{1,64}$/`, `UUID_RE`, `KINDS=
{hacking,development,question}`, `HARNESS_PROFILES={web,device,client,oss}`,
`sock(n)='agent-'+n`, `tmux(n,args)=exec(TMUX,['-L',sock(n),...args])`.

**Auth**: trust upstream Traefik forward-auth — `isAdmin(req)` =
`req.headers['x-auth-role']==='admin'`; `operator(req)` =
`x-auth-email||x-auth-user||'operator'`. `/api/health` is public; everything
else (incl. ws upgrade and vscode proxy) 403s non-admin.

**Activity poller** (1 Hz): for each db agent not dead/paused,
`capture-pane -p -S -800`; on last 15 lines test
`BUSY_RE=/\b[A-Z]\w+ing…|\besc to interrupt\b/` and
`PROMPT_RE=/─{5,}\s*\n\s*[❯>][\s ][^\n]*\n\s*─{5,}/`; extract last user msg
via `/^❯ (\S.*)$/` (skip if line above is `─{5,}`, join `^  \S` continuation
lines, slice 300); persist `last_user_msg` if changed. Build
`ssePayload={activity:{name:{idle,thinking,msg,state}}, questions:[openQs
parsed]}` and push to all `/api/activity` SSE clients.

**Harness scaffold fetch** (`launch=harness`): auth = forward operator's
`pentest_sso` cookie, else `TRIAGE_KEY_<slug>` env, else `TRIAGE_API_KEY`;
`curl -skfSL $TRIAGE_URL/harness.tar.gz?engagement=<p>&profile=<prof> | tar
xz -C $WORKSPACES_DIR`; verify `<WORKSPACES_DIR>/<project>` exists; then
**mintEngagementKey**: `kubectl -n $NS exec deploy/$DEPLOY -- node
server/mint-key.mjs <project> "<project> harness ($HOST)"`, regex out
`/\b[0-9a-f]{48}\b/`, set `process.env.TRIAGE_KEY_<slug>`, append to
`TRIAGE_KEYS_FILE` mode 0600, and substitute `<set-TRIAGE_API_KEY…>` in
`dest/.env` + `dest/.claude/settings.local.json`. Best-effort (warn on fail).

**VS Code web proxy**: per-cwd instance map keyed by
`sha1(dir).slice(0,12)`; pick free port in `CODE_SERVER_PORT_RANGE`; spawn
`$CODE_SERVER_BIN serve-web --host 127.0.0.1 --port P
--without-connection-token --accept-server-license-terms
--server-base-path /vscode/<id> --server-data-dir <dataDir>/<id>
--default-folder <dir> --disable-telemetry`; probe `GET basePath/` until
<500 (40×250ms); HTTP proxy under `app.use('/vscode/:id')` and raw TCP
upgrade replay for websockets. Kill all on process exit.

**Routes**:
- `GET /api/health` → `{ok,host,db}`
- `GET /api/activity` → SSE (`text/event-stream`, `X-Accel-Buffering:no`)
- `GET /api/sessions` → `[{name,windows:1,created,attached,title,auto,kind,
  group,state,cwd}]`
- `GET /api/agents` → raw db rows
- `GET /api/config` → config map
- `GET /api/events?agent&limit`
- `GET /api/questions` → ssePayload.questions
- `POST /api/questions/:id/answer {answer}` → `agent-send`, close q, →running
- `POST /api/questions/:id/dismiss` → close `superseded`, →running if was
  waiting_operator
- `GET /api/claude/projects` → scan `~/.claude/projects/*/`, for each dir
  read newest .jsonl head to recover real cwd, return `[{cwd,count,mtime}]`
- `GET /api/claude/sessions?cwd` → encode cwd (`/._`→`-`), list .jsonl with
  `{id,mtime,size,summary}` where summary = first non-meta user text or
  `type:summary` line (read only first 64KB)
- `GET /api/projects` → enumerate `TARGETS_DIR`+`WORKSPACES_DIR` subdirs +
  `slugs` from `TRIAGE_KEY_*` env keys
- `GET /api/sessions/:name/capture?lines`
- `POST /api/sessions/:name/vscode` → ensure code-web for pane cwd, return url
- `PUT /api/sessions/:name/meta {auto,kind,group,q_default,q_timeout_s}`
- `PUT /api/sessions/:name/state {state∈paused|idle|parked|running, hours}`
- `POST /api/sessions {name,cwd,launch,resumeId,project,profile,kind,group,
  model,auto}` — validate; resolve dir per launch (shell|claude|resume|
  harness); pick model = body||config.default_agent_model; build args for
  `agent-spawn`; `DB.insertAgent(state='spawning')`; exec; on fail →dead.
- `POST /api/sessions/:name/rename` → 400 (name is PK + socket)
- `POST /api/broadcast {text,enter}` → send-keys to every non-dead/paused
- `DELETE /api/sessions/:name?purge=1` → exec `agent-kill`; purge or →dead

**WebSocket `/ws?session&cols&rows`**: verify admin on upgrade; route
`/vscode/*` upgrades to code-web proxy. On connect: `has-session` or close
4404 (and →dead in db if state was non-terminal). `pty.spawn(tmux -L sock
attach-session -t =main, {xterm-256color,cols,rows})`. Binary protocol from
client: `0x01 cols:u16be rows:u16be` = resize; `0x02 dir:u8 n:u8` = scroll
(dir 0=up 1=down) → drive tmux copy-mode (`copy-mode -e` then `send-keys -X
-N n scroll-up|down`; track `weScrolled`, on next keystroke `send-keys -X
cancel` first). Everything else = pty.write. pty.onData → ws.send. pty.onExit
→ re-check has-session → close or 4404. ws close → kill pty. Server-wide
25 s ping/pong heartbeat (`isAlive` flag, terminate on miss).

### `public/index.html` (single file, vanilla JS, ~1800 lines)

GitHub-dark palette (`--bg:#0d1117 --panel:#161b22 --border:#30363d
--fg:#c9d1d9 --accent:#58a6ff`). Inline-SVG favicon (3×3 tile grid).

**Layout**: 44px header (title+host, session count, search input, hint text,
⌘P btn, layout `<select>` 3/6/12/18/24, `matrix` btn, `👁` incognito btn,
`⇶ all` broadcast, `+ new`). `#main` flex-row = `#left` (`#grid` CSS grid,
`--cols/--rows` vars) + `#inbox` right rail (360px, collapses to 0 when
empty). 40px `#dock` strip of parked chips. Floating `#palette`, `#newmenu`,
`#peek` tooltip, `#rain` canvas.

**State (all in localStorage `ai-herder:*`)**: `layout` (maxLive cap),
`order[]` (explicit grid slot order — drag is the only mutator), `fav` Set
(★ = never auto-evict from grid), `pinned` Set (explicitly parked — never
auto-promote), `matrix`, `incog`, `cwd`.

**Grid sizing**: `gridDims(n)` → 1×1..6×4 lookup so 2 tiles fill the screen
instead of sitting in a 3×2 with blanks. `refitAll()` sets --cols/--rows,
font 10px when ≥18 tiles else 12, renumbers `.slot`, and on actual
cols/rows change does `term.reset()` + 800ms quiet window + ws resize msg.

**Tile** (`makeTile`): bar = `[slot][···churn][▸talk ready][.st state
badge][dot][name][★fav][grp pill][info][auto switch][code][▁park][×kill]
[⛶full]`, then `.msg` (last user msg from SSE), then `.term`. xterm with
FitAddon, scrollback 5000, `altClickMovesCursor:false`, theme/font from
`termTheme()/termFont()`. `attachCustomKeyEventHandler`: swallow ⌘P/Ctrl+P
(bubble to doc), Ctrl+` collapses, plain ` collapses (double-tap <400ms =
literal `), Esc collapses when expanded / Shift+Esc passes \x1b through.
`onData` → ws.send (if ws dead while typing: print "…reconnecting…" +
reconnect). Wheel on `.term` (capture, preventDefault): coalesce ±3/event
into one `0x02` frame per 30ms. Bar is drag handle (`dragstart` skips if
target is button/input/contenteditable); whole tile is drop target (use
`relatedTarget` to filter child dragleave); drop = `orderMove(src,this)` +
sortGrid (or parkTile(this)+unpark(src) if src was a chip).

**Streaming policy**: `shouldStream(t)=!document.hidden &&
(!expanded||expanded===t)`. `syncStreams()` (microtask-coalesced)
connects/disconnects accordingly. `connect(t)` opens
`ws(s)://host/ws?session&cols&rows`; onopen set `quiet=now+800` + send
resize; onmessage write + `scrollToBottom()` unless expanded, and if past
quiet window stamp `lastActivity` + (if `SPINNER_RE` matched) `lastSpinner`;
onclose 4404 → refresh, else 2s retry if still wanted.

**Dock chip**: `[dot][grp][name]`, click=unpark, draggable onto tiles,
hover → fetch `/capture?lines=30` into `#peek` (race-guard token,
scrollTop=scrollHeight). `unpark` evicts last non-fav (or last fav if all
faved) when grid full, restores carried activity stamps.

**Activity classes** (500ms timer): tile with ws → `active` if
`now−lastActivity<3s`, `thinking` if `now−lastSpinner<15s`, `idle` if
`lastActivity>0 && now−lastActivity≥3s && now−lastSpinner≥15s`; tile without
ws → use SSE `{thinking,idle}`. CSS: active=green left-border + pulsing slot;
thinking=amber left-border + `···` clip-path dots anim; idle=2px accent
border + glow + `▸ talk` breathe; `asking` (agent in inbox)=2px amber +
pulse. Chips get `.idle`/`.asking` too. Re-run `applySearch()` if query
active.

**SSE consumer**: `EventSource('/api/activity')` →
`{activity:{name:{idle,thinking,msg,state}}, questions:[…]}`. Update tile
`.st` badge + `.msg`, chip `.idle`, then `renderInbox(qs)`: diff qcards Map,
build card = header(agent link, kind pill, "defaults in Ns", × dismiss) +
`<pre>` prompt (scroll to bottom) + option buttons (suggested gets `.sugg`
green; if no options synthesize Enter/Escape/y/n) + free-text input. Toggle
`#inbox.empty` and refit on width change. Mark tiles/chips `.asking`.

**Search** (`/` focuses): substring over `name\ntitle\n<visible viewport
text>` (dump `term.buffer.active` lines). Misses get `.dim` (opacity .12,
grayscale). Enter = jump to first non-dim (clear query first). Red border on
0 hits.

**Command palette** (`⌘P`/`Ctrl+P`, `>`=command mode, Shift opens in `>`):
subsequence fuzzy(`needle,hay`)→`{score,html with <b> wraps}` (gap penalty
+10, lower=better). Session mode lists tiles+chips (idle float top via
score−1000). Command mode: `next idle`, `new session`, `broadcast [text]`,
`matrix`, `incognito`, `layout N`×5, per-group `show|park|only <g>`,
per-session `code|fav|unfav|auto on|auto off|group|kill <name>`. ↑↓ select,
Enter run, click-away/Esc close (clear input on close).

**New-session menu**: project `<select>` (from `/api/projects`, fills
name+cwd on change), name input, cwd input (datalist from
`/api/claude/projects` + projects), segmented radio
shell|claude|resume|harness, model select (''=default,
`claude-opus-4-7`, `claude-sonnet-4-6`), resume `<select size=6>` (lazy-load
`/api/claude/sessions?cwd`, race-guard, dblclick=create), harness sub-panel
(project input + datalist of TRIAGE_KEY slugs, profile select
web|device|client|oss). Create POSTs `/api/sessions`, on success refresh +
unpark + expand. Debounce double-submit.

**Matrix mode**: body class swaps CSS vars to green-on-black; xterm theme →
all-#00ff41 16-slot palette; `.term` gets
`filter:grayscale→sepia→hue-rotate(55deg)→saturate(7)` to crush
truecolor SGR. `#rain` canvas: 14px columns of random katakana+alnum glyphs,
`fillRect(rgba(0,0,0,.08))` fade per frame @50ms; bullet-time: track mouse,
columns within 90px fall at ⅓ speed, spawn expanding ring every ~35px of
travel (max 24, fade by radius/55), 2px hot dot at cursor.

**Incognito**: body class + `term.options.fontFamily='Flow Circular'`
(Google Fonts, preconnected); wait `document.fonts.load` on first toggle
then `refitAll()`.

**Misc**: `toggleExpand(t)` = `position:fixed inset:8px` overlay
(`body.has-expanded #grid{pointer-events:none}`), reset+refit+focus,
syncStreams. `refresh()` every 5s: fetch `/api/sessions`, orderTouch new
names, sort by orderIdx, reconcile tiles/chips/pinned/favs/order, sortDock
(by group then name), sortGrid (only re-append DOM if order actually changed
— appendChild on focused xterm drops focus). Doc-level keydown (skip when in
xterm/contenteditable): `/`→search, `1-9`→expand slot, Tab→unpark first chip.
`visibilitychange`→syncStreams. Group pill: hash name →
`hsl(h%360,45%,55%)`; click=prompt rename, alt-click=park group.

---

## `personas/*.txt`

Each is a roleplay system-prompt for `claude -p` that outputs **exactly one
line** (the nudge). Shared rules: lowercase start, 8-40 words, openers "can
you/lets/I want/great/ok", no em-dashes/bullets, name concrete next step,
shown its own previous nudges (don't repeat), if agent is intentionally
waiting output `__PARK <N>h__`.

- **hacking.txt** — "principal red-team researcher" persona. Seven moves:
  PIVOT (lateral angle on same goal), UNBLOCK (grant bypass for
  license/selinux/pinning/creds), SCOPE/SAFETY (approve-with-boundary or
  redirect to script-only), WRITE IT DOWN (force persistence to
  leads/findings before continuing), SHIP IT (demand triage push / summary),
  KEEP GOING (name the very next step from screen), RESPECT THE LINE (agent
  declines post-exploitation → don't push deeper, redirect to next lead /
  untouched coverage). Hard rules: never destructive on shared infra without
  boundary in same line; on stacktrace name the workaround; on A/B/C pick one
  + 5-10 word why; never generic "continue".
- **development.txt** — "internal-tooling PM" persona. Five moves: FINISH IT,
  TEST IT (local e2e walkthrough), PLAN NEXT (lead with user pain → feature
  sketch, no code yet), UNBLOCK, KEEP GOING. Hard rule: NEVER tell it to
  commit/push/deploy/kubectl — local only.
- **question.txt** — Q&A follow-up. Four moves: DEEPER, VERIFY ("run it and
  paste output"), NEXT, CLARIFY (point at contradiction). 5-30 words.
- **retro.txt** — nightly one-shot auditor (used by `bin/daily-retro`, write
  that script too if you want the cron). Reads
  `notebook/<eng>/reports/.retro-stats.json` + findings/leads/coverage/tasks.
  Writes `retro-<DATE>.md` (≤60 lines: ##Output ##Dead ends ##Gaps ##Tool
  waste ##Tomorrow), updates `coverage/COVERAGE.md` + `tasks/TASKS.md`
  in-place, appends to `/shared/CHEATSHEET.contrib.md` only if a
  grep-generalizable pattern emerged.

---

## `systemd/`

**harness.target** — `Wants=harness-herder.service harness-supervisor.service`,
`WantedBy=multi-user.target`.

**harness-herder.service** — `Type=simple`, `User=ubuntu`,
`EnvironmentFile=/shared/harness/host-config/%l.env` +
`-/shared/state/triage-keys.env`, `Environment=HARNESS_DB=… HARNESS_ROOT=…
TARGETS_DIR=…`, `WorkingDirectory=herder/`. `ExecStartPre` shell: mkdir
`/var/lib/harness`; if db missing but `/shared/state/%l/harness.db` exists,
`PRAGMA wal_checkpoint(TRUNCATE)` it and copy; then
`sqlite3 $HARNESS_DB < schema.sql`. `ExecStart=/usr/bin/node server.mjs`.
`Restart=always`, `RestartSec=3`, **`KillMode=process`** (agent tmux servers
live in this cgroup — control-group would nuke them on restart).

**harness-supervisor.service** — `Type=notify`, `WatchdogSec=120`,
`After=harness-herder.service`, same EnvironmentFile, PATH includes
`harness/bin:~/.local/bin`. `ExecStart=python3 supervisor/supervisor.py`.
`Restart=always`, `KillMode=process`.

**host-config/localhost.env** — `HARNESS_DEFAULT_KIND=hacking`, `PORT=3077`,
commented placeholders for `ANTHROPIC_API_KEY` / Vertex / `TRIAGE_URL` /
`CODE_SERVER_BIN`.

---

## Non-obvious invariants (encode these — they were learned the hard way)

- **One tmux server per agent** (`-L agent-<name>`): a tmux SIGSEGV used to
  take the whole fleet down.
- **`KillMode=process`** on both units: agents are children of herder's
  cgroup; default `control-group` kills every session on a herder restart.
- **DB on local SSD, never NFS**: WAL `-shm` needs coherent mmap.
- **`window-size latest` + `aggressive-resize`**: herder attaches multiple
  ptys at different sizes (grid vs expanded vs stale); without this tmux
  clamps to smallest and you get dot-padding.
- **`prefix None`**: a stray Ctrl-b from the browser can't wedge the pane.
- **Deterministic `--session-id`** in agent-spawn: agents sharing a cwd must
  never `--continue` into each other's thread; respawns must find their own.
- **WS heartbeat 25s ping/pong**: Traefik silently drops idle ws; browser
  keeps `readyState=OPEN` and sinks keystrokes into the void.
- **`term.reset()` on every dimension change** + 800ms `quiet` window:
  xterm reflow + tmux partial redraw leaves garbage; the attach burst must
  not count as activity.
- **`sortGrid()` skips DOM re-append when order unchanged**: `appendChild`
  on a focused xterm's container drops focus, and refresh runs every 5s.
- **Scroll via tmux copy-mode, not xterm scrollback**: tmux attaches in
  alt-screen so xterm has zero local scrollback; xterm's wheel→arrow fallback
  cycles Claude's prompt history instead.
- **agent-send retries once after `C-c`**: pty input buffer fills when the
  agent is mid-stream; drain then retype.
- **Supervisor never kills a live tmux on spawn-timeout**: slow MCP init
  looks like a hang — if `tmux has-session` succeeds, hand off to normal
  classification instead of crash-looping.

Now build it. Write every file, `chmod +x bin/*`, `npm init` the herder with
the four deps, and give me a `README.md` with the install one-liner
(`npm ci && sudo cp systemd/* /etc/systemd/system/ && systemctl enable --now
harness.target`).
