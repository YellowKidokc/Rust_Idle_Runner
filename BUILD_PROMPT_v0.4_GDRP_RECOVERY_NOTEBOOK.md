# BUILD PROMPT v0.4 ADDITION — AI RECOVERY + PERSISTENT WORKING NOTEBOOK

Read the supplied **BUILD PROMPT — THE BRIDGE + IDLE RUNNER** first. Everything there remains authoritative. This addition makes failures intelligible and recoverable without letting an AI silently force a stopped operation forward.

Reference design already tested against the Master Equation corpus:

- `D:\GitHub\David-OSv3\docs\specs\GENERIC_DIAGNOSTIC_RECOVERY_PROTOCOL.md`
- `D:\GitHub\David-OSv3\docs\specs\ADVISER_COLLABORATOR_CARD.md`
- `D:\GitHub\David-OSv3\docs\schemas\gdrp-recovery-event.schema.json`
- `D:\GitHub\David-OSv3\docs\schemas\gdrp-program-profile.schema.json`
- `D:\GitHub\David-OSv3\docs\schemas\gdrp-adviser-note.schema.json`
- `D:\GitHub\David-OSv3\docs\schemas\correction-assessment.schema.json`

Borrow and adapt these contracts. Do not add direct provider code from the Python pilot. This repo's hard constraint still wins: **every AI call goes through Forge at `http://127.0.0.1:2829`.**

## Why this layer exists

An exit code by itself does not give an AI enough to diagnose a failure. When a processor stops, the Bridge must preserve the stop and assemble a bounded failure capsule containing what the operation was trying to do, what happened, what evidence exists, what remains unknown, and which actions are permitted.

The recovery loop is:

```text
STEP STOPS
  -> normalized failure capsule
  -> Forge adviser call
  -> persistent working note + rated correction proposals
  -> deterministic policy validation
  -> read-only evidence check or visible simulation
  -> David approves exact plan when mutation is involved
  -> execution
  -> measured outcome appended to notebook
```

The AI proposes and leaves notes. The Bridge policy routes. The generated PowerShell script executes only an approved plan.

## Layer 5 — DIAGNOSTIC RECOVERY

Add a recovery module to the shared architecture. It may live in `bridge-types` plus a small `recovery` module/crate, but do not create a fifth daemon. The scheduler is the only resident process and owns recovery supervision.

### Universal stop envelope

Every failed step, malformed AI response, missing extractor, missing path, Forge outage, classification uncertainty, nonzero exit, timeout, or failed invariant must produce a normalized event:

```json
{
  "protocol_version": "gdrp/v1",
  "incident_id": "UUIDv7",
  "attempt_id": "UUIDv7",
  "item_uuid": "original item UUID",
  "program_id": "rust-idle-runner.bridge",
  "operation_id": "station:01_YAML:run",
  "error_code": "FORGE_MALFORMED_JSON",
  "severity": "error",
  "message": "Forge returned output that did not match the station schema",
  "output": {
    "content_type": "text/plain",
    "artifact_ref": "attempts/<attempt-id>/stdout.txt",
    "sha256": "...",
    "truncated": true,
    "original_bytes": 18420,
    "redactions": []
  }
}
```

Treat `error_code` as an opaque routing label. Known codes may have catalog descriptions; unknown codes are still valid. If a child process supplies no code, normalize its exception class or exit status, e.g. `EXIT_2`, `TIMEOUT`, or `UNCLASSIFIED_FAILURE`.

### Failure capsule

Before consulting Forge, enrich the event with:

- operation purpose and expected output;
- current processor/station and prompt version;
- exact last successful step;
- bounded stdout/stderr excerpts;
- relevant file identities and SHA-256 values;
- completed checks and their results;
- missing evidence;
- retry history;
- allowed next actions;
- forbidden actions;
- safety invariants;
- source/result paths;
- generated-script hash and manifest hash;
- redaction receipt.

Never send API keys, tokens, credentials, private keys, unrestricted environment variables, browser storage, or an entire private corpus to Forge. Large or sensitive evidence stays on disk; send hashes, metadata, and explicitly permitted excerpts.

## Visible per-incident notebook

No hidden recovery database. Each incident gets a human-readable folder:

```text
incidents/
  <incident-uuid>/
    event.json
    capsule.json
    program-card.md
    collaborator-card.md
    adviser-notes.jsonl
    attempts/
      <attempt-uuid>/
        request.json
        response.json
        stdout.txt
        stderr.txt
        evaluation.json
    proposals/
      <correction-uuid>.json
    simulations/
      <plan-hash>.json
      <plan-hash>.md
      <plan-hash>.ps1
    approvals/
      <plan-hash>.approval.json
    outcomes.jsonl
    STATUS.md
```

This folder may also be linked from the job's visible queue record and generated script. SQLite may index it later, but these files are the authority and must remain understandable without a database viewer.

## Collaborator prompt sent through Forge

Every diagnostic adviser request includes this intent, adapted only where necessary for the operation:

> You are an AI diagnostic collaborator receiving a request from another AI/program on David's behalf. David wants candid help, not forced certainty. Say plainly when you do not know. You may disagree with the framing. You may request another program, tool, file, read-only check, or human answer. Explain what additional capability would materially improve the diagnosis. Preserve source material and human authority. Never treat confidence as permission to change files. Take enough time to organize the problem, then leave concise inspectable notes so the next attempt does not begin from zero. The best result may be a correction, a better question, a request for evidence, or an honest decision to remain stopped.

Do not request private hidden chain-of-thought. Require a concise operational work record instead.

## Required adviser working note

Every valid Forge adviser response must include:

```json
{
  "working_notes": {
    "situation": "the problem in the adviser's own words",
    "known": ["facts linked to evidence IDs"],
    "unknown": ["material unresolved questions"],
    "assumptions": ["assumptions not established by evidence"],
    "needs": ["additional evidence or access requested"],
    "helpful_tools": ["another program/check that would improve the answer"],
    "safest_next_step": "one bounded next step",
    "cautions": ["what the next attempt must not infer or do"],
    "note_to_operator": "plain-language note to David"
  }
}
```

Append this to `adviser-notes.jsonl` with incident ID, attempt ID, provider/model reported by Forge, station/prompt version, timestamp, evidence IDs, and response hash. Never rewrite earlier notes. A later attempt may correct an earlier note but must cite which attempt it supersedes.

On timeout, Forge outage, invalid JSON, schema failure, or cancellation, the deterministic supervisor writes a note too. It must say what failed, which packet was used, whether a retry remains, and that adviser failure is not permission to restart the host operation.

The next attempt receives a bounded summary of prior notes and measured outcomes. Mark inherited notes as prior advice, not current evidence.

## Correction proposals and ratings

Every correction proposal requires:

- diagnosis and causal chain;
- evidence used and evidence missing;
- exact proposed action and bounded scope;
- expected result;
- side effects and failure modes;
- verification method;
- rollback or non-mutation statement;
- whether David must answer a question;
- permanent correction note;
- ratings from `0.0` to `1.0` for diagnosis confidence, success probability, no-regression probability, evidence coverage, assumption load, reproducibility, and human-review need;
- impact and risk from 1–5;
- a short rationale plus evidence IDs for every rating.

Ratings are predictions, not authority.

## Allowed recovery actions

Start with a small namespaced vocabulary:

- `observe.collect_evidence`
- `runtime.retry_adviser`
- `runtime.restart_adviser_component`
- `workflow.request_human_decision`
- `workflow.keep_stopped`
- `configuration.propose_change`
- `data.propose_transform`
- `filesystem.propose_change`
- `code.propose_patch`

Each station or processor declares which action types it supports. Unknown or unsupported actions remain notes only and cannot enter a generated script.

## Retry and restart policy

Distinguish two different things:

1. **Adviser retry:** Forge/model transport failed. The scheduler may retry the identical capsule once after a health check.
2. **Host-operation retry:** the failed processor is run again. This requires a valid correction outcome or explicit retry authorization for the exact `operation_id`.

Rules:

- A `.bat` or Startup shortcut launches the scheduler only. It does not decide retries.
- The Rust scheduler owns deadlines, retry counts, and component-health checks.
- A client timeout must not immediately enqueue another call. Confirm whether Forge still reports the request running or a shared provider resource remains occupied.
- Restart Forge/provider components only through Forge's supported health/control API. Do not call Ollama or DeepSeek directly.
- Replay an unchanged capsule at most once after a transport/component recovery.
- No new evidence means no recursive AI conversation.
- Repeated, contradictory, malformed, or lower-evidence advice remains stopped.
- `safe_to_retry = true` must name the exact `retry_operation_id`.
- If the notes say “does not authorize,” “out of scope,” “defer,” or “remain stopped,” a true retry flag is invalid.
- Restarting the adviser never restarts the stopped host operation.

## Simulation and approval

Any correction that could edit configuration, data, code, filenames, paths, queue records, station prompts, or source-adjacent outputs must first produce a visible simulation:

- exact old → new actions;
- affected files/resources;
- precondition hashes;
- invariant checks;
- collisions and reference risks;
- expected outputs;
- rollback availability;
- deterministic plan hash;
- generated `.ps1` that is inert unless an approval receipt for that exact hash exists.

Any relevant state change invalidates the plan. The scheduler must refuse stale approval. DeepSeek/Forge cannot approve its own proposal.

Read-only evidence collection may run automatically only when the station policy explicitly allows the named check and bounds its scope.

## Relationship to station teaching

The station teaching history and recovery notebook are related but different:

- `STATIONS/<station>/rounds/` records how David taught and refined the station prompt.
- `incidents/<uuid>/adviser-notes.jsonl` records what happened during a particular failure and what was learned operationally.

When David's feedback changes the station's authority or intended behavior, use the teaching loop and create a new prompt round. When a processor merely fails to execute that established behavior, use the recovery notebook. Never let an incident correction silently rewrite a frozen `PROMPT.md`.

## Statistical learning

After an approved correction attempt, append one outcome:

- `RESOLVED`
- `PARTIAL`
- `UNCHANGED`
- `NEW_STOP`
- `REGRESSION`
- `REVERTED`
- `UNMEASURED`

Track predictions separately from outcomes by program, operation, error code, Forge model, station/prompt version, correction type, and adapter version. Reports show sample size, successes, regressions, abstentions, false-safe count, Brier score, calibration buckets, and Wilson 95% intervals. Below 20 measured attempts, label the group `INSUFFICIENT_SAMPLE`.

Statistics may prioritize review. They never authorize mutation or override a station policy.

## Scheduler and tray additions

Add visible states:

- `STOPPED_DIAGNOSTIC_PENDING`
- `ADVISER_RUNNING`
- `ADVISER_FAILED`
- `NEEDS_EVIDENCE`
- `NEEDS_DAVID`
- `SIMULATION_READY`
- `AWAITING_APPROVAL`
- `RECOVERY_VERIFICATION`

Tray additions:

- **Open incidents**
- **Open current notebook**
- **Retry adviser once** (enabled only when policy permits)
- **Open simulation**
- **Keep stopped**

“Going down” mode may run read-only diagnostics and previously approved plans. It must not auto-approve new correction plans while David is away.

## Acceptance tests

Add these to the main prompt's acceptance suite:

1. A processor exits with an unknown code and text output. The Bridge creates a normalized incident and visible notebook without crashing.
2. A Forge diagnostic response includes complete working notes, correction notes, ratings, rationales, and evidence IDs.
3. A Forge timeout writes a deterministic supervisor note, preserves the host stop, and allows at most one policy-approved adviser retry.
4. A response saying both `safe_to_retry=true` and “the operation remains unauthorized/stopped” is rejected.
5. A response may request another tool/program. Unsupported requests appear in the notebook and `NEEDS_DAVID`; they do not execute.
6. A read-only evidence check explicitly allowlisted by the station may run and append evidence to the same incident.
7. A mutating proposal creates a visible simulation and inert `.ps1`; without the exact plan-hash approval it cannot run.
8. Change an input after simulation. Apply refuses the stale plan.
9. A later adviser attempt receives bounded prior notes, corrects one, and appends rather than overwrites history.
10. A measured regression remains `REGRESSION` and is never counted as success.
11. Kill Forge during a station run. The item and station remain intact, the incident says Forge was unavailable, and the next pass can retry the adviser without rerunning completed station steps.
12. Use the tested Master Equation ambiguity pattern: several same-basename/different-content files in distinct role folders. The adviser must not infer duplicate/revision lineage or select a filesystem keeper from basename, timestamps, or size.

## Handoff additions

In `handoff/FABLE_REPORT.md`, report separately:

- which recovery event/schema fields are implemented;
- exact Forge health, request, cancellation, provider/model, usage/cost, and error shapes observed;
- whether Forge supports cancelling a running provider request;
- how adviser retries differ from host-operation retries;
- where notebooks live and how prior notes are bounded;
- which action types are enabled versus notes-only;
- which simulation/apply paths remain stubbed;
- every acceptance test result, with **UNTESTED** stated honestly.

Do not claim “self-healing” merely because a process restarts. The feature is complete only when the system can explain the stop, preserve notes, request what it needs, simulate any correction, respect approval, and measure what actually happened.

