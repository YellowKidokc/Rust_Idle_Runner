# BUILD PROMPT â€” Rust Idle Runner
POF 2828 | Oct 19, 2026 | v0.1 â€” edit freely before handing off

---

You are building **Rust Idle Runner**: a small, always-on Windows program, written in Rust, that runs background jobs automatically when the computer is idle, at night, or on schedule. Any project on this machine can plug into it by dropping in a job file. The runner itself should almost never need to change â€” all customization lives in job files.

## Context
- Owner: David. Windows 11. Projects live under `D:\GitHub\`.
- An existing Rust app, `D:\GitHub\nerve`, already has working, tested idle-detection code in `source\src\idle.rs`. **Reuse its patterns**: `GetLastInputInfo` idle detection, the 5-second poll, the cancel flag set on user return, the on-disk lock with stale-lock clearing, atomic writes (temp + rename), and append-only JSONL receipts. Read that file before you start.
- Do NOT modify nerve. This is a new repo: `D:\GitHub\Rust_Idle_Runner`.

## What it must do

### 1. Run as a background program
- A single `.exe`, started at Windows login (offer an install command that registers it in the Startup folder or Task Scheduler â€” no admin required).
- A tray icon with a menu: **Status**, **Pause all**, **Resume**, **Run next job now**, **Open today's report**, **Open logs folder**, **Reload jobs**, **Quit**.
- Tray icon state: idle-waiting / running a job / paused / away / error.

### 1b. Away mode â€” "I'm going down for a while" (IMPORTANT)
David has no fixed schedule, so clock-based "night" windows are unreliable. He tells the runner instead.
- Tray menu (and a left-click quick menu) has **Going away â–¸ 1h / 2h / 4h / 8h / Until I'm back**, plus **I'm back**.
- Also expose it as a command-line flag (`rust-idle-runner.exe --away 4h`, `--back`) so he can pin a shortcut to the taskbar / Start, or bind a hotkey.
- While away mode is active:
  - Jobs start **immediately** â€” no waiting for the idle threshold.
  - Jobs with `window = "away"` (the heavy ones) become eligible. `window = "night"` jobs are also eligible.
  - A brief mouse bump does NOT stop work. Stop only if there is **sustained input for 2+ minutes** or he clicks **I'm back**.
  - Time caps can be longer: a job may declare `away_max_minutes`, used instead of `max_minutes`.
- Away mode ends at its timer, on **I'm back**, or on sustained input. It persists across a reboot only until its end time.
- Show the remaining away time in the tray tooltip.

### 2. Discover jobs
- Config file: `%APPDATA%\RustIdleRunner\config.toml` with a list of **registered roots** (e.g. `D:\GitHub\deep-research-hub`, `C:\Users\David\Documents\faiththruphysics.com\Canon\DEBATE_MAP`).
- On start and on "Reload jobs", scan each root for files named `*.job.toml` (any depth, skip `.git`, `node_modules`, `target`, `__pycache__`, `.venv`).
- Each job file describes ONE job. A project plugs in by adding a job file. Nothing else.

### 3. Job file format (`*.job.toml`)
```toml
id          = "case-agent-p03"          # unique across all roots
name        = "Case agent â€” P03 Life"
enabled     = true
command     = "python"                  # any executable
args        = ["case_agent.py", "--part", "P03"]
working_dir = "D:/GitHub/deep-research-hub/apps/case-agent"   # defaults to the job file's folder
env         = { OLLAMA_HOST = "http://localhost:11434" }      # optional

[when]
trigger      = "idle"      # "idle" | "schedule" | "both"
idle_minutes = 30          # used when trigger includes idle
schedule     = "daily 02:00"   # simple forms: "daily HH:MM", "hourly", "every 6h", "weekly sun 03:00"
window       = "any"       # "any" | "night" (22:00â€“08:00) | "HH:MM-HH:MM"

[limits]
max_minutes   = 90         # hard time cap per run
cooldown      = "6h"       # min time between successful runs ("0" = no cooldown)
repeat        = "until_done"   # "once" | "until_done" | "always"
priority      = 50         # higher runs first
stop_on_user_return = true # if false, job keeps running when David comes back (e.g. pure downloads)
paid          = false      # true = this job spends money (DeepSeek/Claude API); see budget below
resources     = ["ollama"] # shared resources this job needs; see Resource locks
```

### 3b. Resource locks
Some engines can only safely serve one heavy job at a time. Ollama is a single local server; two jobs that each load a large model will fight over GPU memory and evict each other's models. So:
- A job lists the shared `resources` it uses: `"ollama"`, `"gpu"`, `"deepseek"`, `"claude"`, `"network-heavy"`, or any custom name.
- `config.toml` sets a limit per resource, e.g. `ollama = 1`, `deepseek = 3`, `network-heavy = 2`. Unlisted resources default to 1.
- With `max_concurrent` > 1, two jobs may run together only if every shared resource stays within its limit. A downloader and an Ollama job can run side by side; two Ollama jobs cannot.
- Before starting an `ollama` job, check `http://localhost:11434` responds; if not, try starting `ollama serve` once, else skip the job with outcome `SkippedResource`.

### 4. Job contract (how jobs talk back)
Jobs are ordinary programs. The runner communicates with them through:
- **Env vars passed in:** `IDLE_RUNNER_JOB_ID`, `IDLE_RUNNER_RUN_ID`, `IDLE_RUNNER_STATE_DIR` (a per-job folder the job may write resume state into), `IDLE_RUNNER_STOP_FILE` (path of a file the runner creates when the job should stop gracefully).
- **Graceful stop:** when David returns (and `stop_on_user_return = true`) or `max_minutes` is hit, create the stop file, wait up to 60 s for the process to exit, then kill the process tree.
- **Exit codes:** `0` = success, more work remains â†’ eligible again after cooldown. `10` = success, job is DONE (for `repeat = "until_done"`, stop scheduling it until its job file changes). Anything else = failure.
- **Output:** capture stdout/stderr to `logs\<job-id>\<run-id>.log`. If the job prints a final line starting `RESULT:` (JSON), store it in the receipt.

### 5. Scheduling behaviour
- Default `max_concurrent = 2`, governed by resource locks (3b) — e.g. a download plus one Ollama job.
- While the machine stays idle, keep pulling the next eligible job â€” don't stop after one run. Ordering: priority, then longest-since-last-run.
- Eligible = enabled, trigger satisfied, inside window, cooldown passed, not marked done, budget allows.
- "Run next job now" from the tray ignores idle/window/cooldown but still respects budget.
- Pause-all persists across restarts until resumed.

### 6. Budget guard for paid jobs
- `config.toml` holds `daily_paid_minutes` (default 120). Paid jobs stop being scheduled once the day's total paid run-time is reached. Reset at midnight.
- Never put API keys in job files or logs. Jobs read their own keys from their own `.env`.

### 7. Records
- `receipts\receipts.jsonl` â€” one line per run: job id, run id, start/end, outcome (`Complete`, `Done`, `UserReturned`, `TimedOut`, `Failed`, `SkippedBudget`), exit code, idle minutes at start, `RESULT` payload.
- `reports\report-YYYY-MM-DD.md` â€” human morning report: what ran, how long, outcome, RESULT summaries, failures with the last 20 log lines. Rebuildable from receipts.
- `state.json` â€” last run per job, done flags, pause flag. Atomic writes only.
- Everything under `%APPDATA%\RustIdleRunner\`.

### 8. Safety rules
- The runner never deletes, moves, or edits files outside its own `%APPDATA%\RustIdleRunner\` folder. What jobs do is their own business.
- Single-instance: a second launch just focuses the existing tray.
- One bad job file must not stop the others â€” log the parse error in the report and skip it.
- If a job crashes three runs in a row, auto-disable it and flag it in the report.

## Deliverables
1. Repo `D:\GitHub\Rust_Idle_Runner` with `Cargo.toml`, `src\` split by concern (`idle.rs`, `jobs.rs` (discovery + parsing), `scheduler.rs`, `runner.rs` (process spawn/stop), `tray.rs`, `report.rs`, `config.rs`, `main.rs`).
2. `README.md` written for David: what it does, install, how to add a job in 3 steps, how to read the report.
3. `examples\` with three job files:
   - `hello.job.toml` â€” runs a script that sleeps 2 min, honours the stop file, exits 0. For testing.
   - `transcripts.job.toml` â€” runs the YouTube transcript downloader (`D:\GitHub\Research-Acquisition\yt-transcript-downloader`), trigger `both`, `stop_on_user_return = false`, not paid.
   - `case-agent.job.toml` â€” placeholder for `case_agent.py` (not built yet), `enabled = false`, `paid = true`.
4. `install.ps1` (register startup, create config with default roots) and `uninstall.ps1`.
5. Tests: job-file parsing (good + bad files), eligibility logic (cooldown, window, done, budget), receipt writing, graceful-stop then kill.

## Acceptance checks (run these before saying you're done)
- `cargo build --release` and `cargo test` pass.
- With `hello.job.toml` registered and `idle_minutes = 1`: leave the machine alone â†’ job starts within ~70 s; move the mouse â†’ job receives the stop file and exits; receipt says `UserReturned`.
- A malformed job file appears as an error in the report; other jobs still run.
- Report the honest status of each acceptance check â€” mark anything untested as **UNTESTED**.

## Build order
Do it in this order and stop for review after step 3:
1. Config + job-file discovery/parsing + tests.
2. Eligibility/scheduler logic + tests (no processes yet).
3. Process runner with stop file / timeout / kill + receipts. **â† stop here, show David**
4. Idle loop (port from nerve) + tray.
5. Report, budget, auto-disable, install scripts.

