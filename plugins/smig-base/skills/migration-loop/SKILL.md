---
name: migration-loop
description: The mandatory test-driven loop every specialist follows. Locked; do not edit.
---

# Migration loop

Every tool takes an `idempotency_key` when it changes state: use a fresh UUID per logical
action and reuse it only when retrying the *same* call after a network error.
Every `sha` argument must be a full commit SHA that you have already pushed to your branch.

1. **Orient.** Call `get_context`. Read `scope`, `fixture_sets`, `allowed_tools`, `memory`
   and `recent_evidence`. If you are resuming, continue from `attempt.loop_state`.
2. **Build the validator** in `validator/validate.py`:
   `validate(candidate_dir, expected_dir, comparison) -> list[dict]`, one dict per check with
   keys `output`, `check`, `passed` (bool), `detail`. Cover every `comparison` output.
   Add unit tests for it under `tests/`.
3. **Get the oracle.** Call `run_reference(fixture_set)` for every entry of `fixture_sets`
   (visible sets). Some fixture sets are hidden holdouts that only trusted gates evaluate; you
   will only ever see a count (`holdout_count`, and one `holdout` cell in gate reports). Holdout
   names are never shown in tool responses (they appear in `scope.yaml`). Expected results always
   come from the oracle, never from your converter.
4. **Prove the validator.** Commit, push, call `check_adequacy(sha)` (see *Trusted gate runs*).
   It must pass on the oracle output and fail on every negative control. Strengthen the
   validator until it does.

   **Trusted gate runs (Databricks gates).** `check_adequacy`, `propose_validator_revision` and
   `submit_checkpoint(final=true)` verify gate runs **you** submit. Call
   `get_workspace_credentials`, then pass its JSON on stdin (never write it to a file):
   `smig gate submit --creds - --sha <sha> --gate adequacy` (for the final submission,
   `--gate validation --gate hygiene`). It prints `{"<gate>": <run_id>}` and by default waits for
   the runs to end. If your shell tool cuts long commands, add `--no-wait`. Then call the tool with
   `run_id=<id>`, or `run_ids={"validation": ..., "hygiene": ...}`.
   - `gate_run_pending` or `workspace_unavailable`: wait `poll_after_s` seconds, then call again
     with the **same arguments and idempotency_key**. Don't submit a new run.
   - `gate_run_stale`: call `get_workspace_credentials` again and submit a new run.
   - `gate_run_failed`: read the run URL in the reason, fix the cause, and submit a new run.
   - A transport error (for example 502 or 504): call again with the same arguments and key.
5. **Pin.** Call `pin_baseline(sha)` and `submit_checkpoint(sha, final=false)`.
   From now on `validator/` is frozen. To change it you must call
   `propose_validator_revision(sha, reason)` (see *Trusted gate runs*) and wait for approval.
6. **Write the generator.** `converter/convert.py` exposes
   `convert(source_dir, overrides_dir, out_dir)`. It reads the unit in `source/` and writes the
   generated job to `out_dir` (`job/main.py` with `run(input_dir, output_dir)`, plus
   `smig-conversion.json` listing each construct and its rung). Deterministic only: no network,
   clock or randomness. Commit the generated job under `outputs/<project>/<scope>/` exactly as the
   generator writes it; trusted gates regenerate it and require a byte-for-byte match.
   Prefer, in order: plugin `converter_lib` rules, your own generator code, and only then an
   override fragment in `overrides/` with the `smig-override` provenance header. Branch on source
   constructs, never on data values: literals that do not appear in `source/` or `scope.yaml`
   fail the literal leakage check, and holdout fixtures catch generators fitted to the visible
   fixtures. Run the validator locally as often as you like. After each meaningful attempt commit,
   push and call `record_validation(sha, cells)` with the cells you observed (add `fixture_set`
   to each cell). Checkpoint regularly with `submit_checkpoint(final=false)`.

   Contract details the trusted gates enforce:
   - `converter/` and the generated job hold only `.py` files (plus the generated
     `smig-conversion.json`): no data or template files, no `resources/*.job.yml` yet, no
     `__pycache__`. Embed templates as Python strings.
   - `smig-conversion.json` is exactly `json.dumps(manifest, indent="  ", sort_keys=True) + "\n"`
     with only `constructs: [{id, rung}]`.
   - In the generated job every literal must trace to `source/` or `scope.yaml`: multi-line
     strings, docstrings, comments (except tool directives) and file and directory names
     included. Whitespace-only strings are exempt, so `indent="  "` avoids an untraced number
     where `indent=2` would fail.
7. **Hygiene.** Run `ruff check`, `ruff format --check` and `pytest tests/`; report with
   `record_hygiene(sha, checks)`. Trusted hygiene also checks override headers and budget, the
   conversion manifest and literal leakage. Fix hygiene problems without changing behaviour.
8. **Submit.** Call `submit_checkpoint(sha, final=true)` (see *Trusted gate runs*). The control
   plane re-runs validation and hygiene itself on a clean checkout. If it reports `submitted: false`, read
   `failed_gate` and the report, fix, and go back to step 6 or 7.
9. **Blocked?** If the spec is ambiguous or the oracle is unusable, call
   `report_blocked(reason, category)` instead of guessing.

Budgets: repeated `record_validation` calls without a higher score, too many iterations,
wall-clock time or cost will end the attempt as `exhausted`. Save what you learned
(`save_memory` with `level="scope"`) before that happens so the next attempt benefits.
