# BUILD PROMPT v0.3 ADDITION: The Task Queue

Read BUILD_PROMPT.md first. Everything there still holds. This adds one feature: David can queue AI work in plain words, and the runner does it while he's away.

## Why
v0.2 runs jobs, but every job must be a program someone already wrote. David wants to hand over a task the way he'd hand it to a person: "canonize the Master Equation folder with DeepSeek, put the results here." The queue turns that note into work.

## 1. Task files
- Queue folder: `%APPDATA%\RustIdleRunner\queue\` (plus any registered root's `queue\` folder).
- One task = one Markdown file, `*.task.md`. Front matter, then the instructions in plain words:

```markdown
---
id: canon-master-eq-01
model: deepseek-chat          # or ollama:qwen3:4b-instruct, ollama:gemma4:latest, deepseek-reasoner, claude
inputs:                       # files or folders the AI reads
  - C:/theophysics/OPUS/MASTER_EQUATION
per_file: true                # true = run the instructions once per input file; false = all inputs at once
output: C:/theophysics/OPUS/MASTER_EQUATION/_CANON_RUN
parallel: 30                  # max at once for API models; Ollama is always 1
max_minutes: 120
paid: true
---
For each file: state its core claim in one sentence, list every
equation it uses, mark each as defined / derived / asserted, and
flag anything that conflicts with the nine-coordinate canon.
```

- A template file `_TEMPLATE.task.md` lives in the queue folder so David can copy it.

## 2. How a task runs
- Tasks are processed in queue order (oldest first), as ordinary jobs under the v0.2 rules: away mode, idle triggers, budget, resource locks, receipts, and the morning report all apply.
- **All AI calls go through Forge** at `http://127.0.0.1:2829` (David's single AI-call service, repo Rust_AI_API). The runner never holds API keys. If Forge is not responding, the task waits with outcome `SkippedResource` and the report says so.
- `per_file: true` sends one request per input file, up to `parallel` at once (Ollama: 1). Each result is saved as `<output>/<input-name>.result.md`. A file that already has a result is skipped, so an interrupted task resumes where it stopped.
- When every input has a result, write `<output>/_TASK_SUMMARY.md` (counts, failures, time, cost if Forge reports it) and move the task file to `queue\done\`.
- A failed file is retried once, then listed as failed in the summary. It never stops the rest.

## 3. Check before running, not at 3 a.m.
- When a task file appears, validate it immediately: front matter parses, inputs exist, output folder is writable, and **the model actually exists** (ask Ollama's `/api/tags` for local models; ask Forge for API models). A task with a bad model or missing input goes to `queue\needs_attention\` with a one-line reason, and the tray shows a warning. (The old Overnight Crew never ran because it named a model that wasn't installed. This check prevents that.)

## 4. Tray additions
- **Add task**: opens the queue folder with the template selected.
- **Queue** submenu: shows waiting tasks, the running task with "file 37 of 180," and lets David move a task to the top or pause it.
- The away-mode menu shows how many queued tasks are waiting, so David knows there's work ready before he leaves.

## 5. Acceptance (add to the v0.2 checks)
- A task over a folder of 5 small text files with `ollama:qwen3:4b-instruct` produces 5 result files and a summary while away mode is on.
- Kill the runner halfway; restart; the task resumes and does not redo finished files.
- A task naming a model that isn't installed lands in `needs_attention` within seconds of being dropped in, with the reason.
- With Forge stopped, an API task waits and the report says Forge was down.
- `parallel: 30` on a DeepSeek task runs up to 30 requests at once (verify in the logs), and Ollama tasks never run more than one at a time.

## Build order
Add the queue after v0.2 step 5. The v0.2 review stop after step 3 still applies.
