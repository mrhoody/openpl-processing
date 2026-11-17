# Powerlifting Singapore batch workflow

Use this procedure when asked to process or submit PS results, resume a batch, or monitor and repair its CI. Creating or editing this setup does not invoke processing or submission.

## Scope and local state

This GitHub repository stores the workflow. Inputs live in ignored `data/`; checkpoints and intermediate exports live in ignored `.workflow-state/`. The sibling `../open-powerlifting-database-maintenance` checkout holds the data branch. Submit that branch through its existing GitLab fork, `hud.syafiq/open-powerlifting-database-maintenance`, to the original `openpowerlifting/opl-data` project on GitLab. Never submit data through a GitHub PR or push directly to the original project.

Represent each event as **one meet, one platform record, and one meet folder**, even when its source spans multiple days, platforms, or sessions. Consolidate differently structured inputs into that event record rather than creating separate meets. Retain the actual source day/platform coverage as provenance in the checkpoint and originals; normalization must not discard results or falsify source evidence.

Keep a batch checkpoint in `.workflow-state/` with:

- Input filenames and fingerprints, meet/date/day/platform coverage, evidence and export paths, destination meet directories, and ready/held/submitted status.
- Data checkout, branch, freshly fetched upstream default branch and tip, fork and target project, MR URL, and submitted head commit.
- Validation commands/results, pipeline ID/URL and associated commit, required job results, and unresolved decisions with their evidence.

Update the checkpoint after each completed step and before handoff. On resume, inspect the actual branch, files, MR, and pipelines; reconcile them with the checkpoint instead of trusting cached status. Keep credentials in the existing credential manager or environment, outside tracked files and checkpoints. Preserve source files. This workflow installs no global skills, hooks, scheduled jobs, parent-folder instructions, or global agent configuration. Install a missing Rust toolchain only as described in step 4.

## 1. Synchronize upstream — mandatory gate

Run this step first on **every invocation, including resumed and CI-monitoring runs**, and repeat it before each repair or submission cycle. A saved upstream tip or a clean CI result never replaces a fresh synchronization.

1. In the sibling checkout, inspect `git status --short --branch`, the current branch, local batch branches, and remotes. Identify any existing batch and submission before creating another. If tracked changes, relevant untracked work, an unfinished Git operation, or conflicts prevent safe synchronization, stop and ask the user to resolve them. Preserve all work; do not automatically stash, discard changes, reset, or force-push.
2. Discover the original repository's current default branch with `git ls-remote --symref https://gitlab.com/openpowerlifting/opl-data.git HEAD`. Fetch that advertised branch directly from the original URL with `git fetch https://gitlab.com/openpowerlifting/opl-data.git refs/heads/BRANCH`, replacing `BRANCH` with the discovered name. Immediately record `git rev-parse FETCH_HEAD` as the upstream tip. A fetch from the fork alone is insufficient. If discovery or fetch fails, stop; do not use stale refs.
3. For a new batch, create a uniquely named `ps/<batch>` branch from that exact fetched tip. For an existing batch, switch to its recorded branch and merge that fetched tip into it, preserving batch commits. If the branch exists only on the fork, fetch and resume it rather than starting a replacement. Stop on conflicts and request resolution; leave the work intact.
4. Require `git merge-base --is-ancestor UPSTREAM_TIP HEAD` to succeed, using the freshly recorded SHA. Record the successful gate in the checkpoint. Until it succeeds, do not convert, edit, repair, validate for submission, or push data. Repeat validation after any upstream merge that changes the branch, including the Rust checker provenance and toolchain checks in step 4.

## 2. Inventory inputs and prevent duplicates

List every file in `data/` and map it to its event, source date/day/platform/session, and single destination meet folder in the checkpoint. Inspect actual source contents, not just filenames. Compare against the freshly synchronized `meet-data/ps`, existing fork branches, and open or merged upstream submissions, including the batch MR. Match event metadata and all source results coverage; different filenames, day/platform splits, or meet IDs do not prove distinct events. Reuse an existing submission for the same batch and an existing event record where possible. Skip already submitted results; treat corrections or consolidation of existing split records as evidence-backed updates rather than additional duplicate meets. If submission history cannot be checked, stop before creating a potentially duplicate submission.

Present a completeness report for every in-scope meet, including mapped existing PS folders: identify the meet/folder, files present, missing or unusable source files, missing days/platforms/sessions, and what the user must supply or confirm. Distinguish missing sources from `meet.csv` or `entries.csv` awaiting normal conversion. Where expected coverage is unknown, mark it unverified rather than complete; folder numbering alone does not establish a missing meet. Highlight gaps to the user immediately and keep the report in the checkpoint, not just in internal notes.

Confirm that each event has final results for every expected day, platform, and session before consolidating it. Hold incomplete or unverified events and record precisely what is missing; independently complete events may proceed. A single day/platform backup does not establish that a multi-day or multi-platform event is complete. If nothing is ready, report the held inputs and stop without an MR.

For `.openlifter` backups, load a copy through OpenLifter and export its OpenPowerlifting CSV. Keep the backup and export mapping locally, verifying all days/platforms against the backup and other source evidence. If OpenLifter is unavailable or the export cannot be verified, hold that input and request assistance; do not invent a backup parser or fabricate missing results.

## 3. Process and reconcile

Read the fetched upstream documentation, nearby PS meets, and relevant tool usage before conversion. Follow the PS layout: one `meet-data/ps/YYNN/` directory per event, containing:

- `meet.csv`: one event-level metadata record, matching the existing PS schema, with `Date` set to the verified event start date. Use the event name rather than separate day/platform titles. `MeetName` must not contain a full four-digit year; if the name includes a year, shorten it (for example, `National Championships 2026` becomes `National Championships 26`). Keep the full year in `Date` and preserve original source CSVs unchanged.
- `original.csv`: the unchanged source CSV, retained beside the processed data. For multiple source days/platforms/sessions, retain every original with a distinct suffix, following PS precedents such as `original-D1AM.csv`. Record each source's mapping into the consolidated results.
- `entries.csv`: one final processed dataset combining all verified days, platforms, and sessions of that event into its single meet/platform record.

`YY` is the meet year's last two digits; `NN` is the zero-padded meet sequence for that year, starting at `01` (for example, `2601` is the first meet of 2026, not January). Each event gets one number, not one per source day/platform. Allocate new meets in chronological event-start-date order, not input filename, processing, or submission order. Check existing meets and pending submissions before assigning numbers and record the mapping in the checkpoint. Preserve published meet IDs, including historical ordering exceptions; if fitting a new meet chronologically would require renumbering existing IDs, or the ordering is ambiguous, ask the user before assigning it.

Reuse upstream tools only for compatible source formats: `scripts/openlifter-convert` consumes a full OpenLifter `original.csv` export in the current meet directory and produces `meet.csv` and `entries.csv`. For a single-source event, run it only in the intended new meet directory after copying the verified export there as source evidence. For multi-source events, convert each source separately in local working directories before consolidating; avoid overwriting earlier converted results or originals. For other formats, follow the corresponding existing upstream conversion pattern or request clarification. Keep private backups and working artifacts local; include only source evidence appropriate for public submission under upstream conventions.

Reconcile converted metadata, entry counts, identities, divisions, equipment, bodyweights, attempts, totals, and placings against every source for the event. Account for every source result without duplicating or dropping entries during consolidation, preserving legitimate repeated entries and source-supported event-wide placings. Resolve conflicting session metadata or rankings with evidence or the user rather than silently concatenating incompatible records. Retain an evidence trail for corrections. Ask the user about uncertain results or identities; hold affected meets until resolved rather than guessing or deleting troublesome entries. Change a shared identity registry only when necessary for these PS entries and supported by evidence; never overwrite another lifter's identity to silence a checker error.

### Missing or invalid birthdates

Keep the `BirthDate` column in `entries.csv` even when its cells are blank; upstream requires the column for meets since 2020, not a date for every lifter. Normalize unambiguous formatting or correct a date only from reliable evidence. If the source birthdate is missing, malformed, impossible, or demonstrably implausible and no verified correction exists, leave that processed cell blank. Preserve the original CSV unchanged and record the affected row, reason, and action in the checkpoint; never invent a replacement birthday.

Preserve independently reliable `Age` or `BirthYear` values. Leave them blank too when their only basis is the invalid birthdate; do not retain ages calculated from bad dates. Keep the recorded `Division`, mapping its label only through established PS conventions, and use the fetched `meet-data/ps/CONFIG.toml` division bounds via the upstream checker to constrain unknown age. Do not manufacture an exact age or birth year from a division range, infer a narrow range from Open, or add custom inference code.

A blank birthdate alone is not an incomplete-meet blocker when source coverage is complete and upstream checks pass. If credible birthdate/age evidence conflicts with the recorded division, or identity remains ambiguous, ask the user rather than clearing valid evidence or changing the division merely to pass checks.

## 4. Validate data and enforce scope

For every ready event, verify that all its source days/platforms/sessions are consolidated into exactly one meet/platform record and one folder; that the folder contains the metadata, all original source CSV(s), and final entries; that `MeetName` follows the naming rule above; that the folder year matches the meet date; and that new meet numbers follow the agreed chronological mapping without collisions. Presence alone is insufficient: flag unreadable, empty, header-only, malformed, or truncated required files and incomplete results coverage. Refresh the completeness report with exact affected paths and hold unresolved events out of submission.

In the data checkout, run the fetched upstream checks, including the Rust checker below. Use the current upstream `Makefile`, `tests/`, and `.gitlab-ci.yml` as the source of truth for additional required checks. Verify birthdate handling against the rule above, including the retained column, independent age evidence, and configured division bounds. A missing dependency, failed check, or unavailable check is a blocker to claiming validation, not a pass.

### Rust checker — required and matched to upstream

1. After every synchronization, verify checker build inputs match the freshly fetched **original OpenPL repository**, not just the fork or a saved local revision. From the data checkout, require `git diff --exit-code UPSTREAM_TIP -- checker/ crates/ Cargo.toml Cargo.lock rust-toolchain.toml .cargo/ tests/` to pass, replacing `UPSTREAM_TIP` with the recorded SHA. Inspect untracked source/build overrides too. Stop on unexplained differences; preserve local work rather than overwriting it. A version string alone does not establish a match.
2. Read the fetched `rust-toolchain.toml` and check `rustc --version`, `cargo --version`, and `rustup show active-toolchain` from the data checkout. If Rust/Cargo or rustup is unavailable, install it using the [official rustup instructions](https://rust-lang.org/tools/install/), then install the repository-pinned toolchain if missing. Respect installation permissions and preserve unrelated user settings. Ensure the active compiler matches the upstream pin, including any required components/targets; environment or directory overrides must not silently select another toolchain. Do not substitute an arbitrary latest/stable compiler or edit the upstream pin/lockfile to make compilation work. If installation, toolchain verification, or dependency retrieval fails, report the blocker instead of skipping Rust validation.
3. Build and run from the synchronized data checkout using Cargo, so changed upstream source is rebuilt and dependencies stay locked. Do not substitute a globally installed `checker`, an old binary invoked directly, or an unverified prebuilt image. The current commands, run from the checkout root, are:

   ```sh
   cargo run --locked --release --bin checker -- --project-root . --timing
   tests/check-lifter-data
   tests/check-line-endings
   tests/check-file-sizes
   ```

   Run any further checks required by the fetched CI configuration. A targeted PS check may help during repairs but does not replace the full required checker run before submission.
4. Record the upstream SHA, data-branch commit, source-match result, active toolchain/compiler/Cargo versions, commands, and outcomes in `.workflow-state/`. Reverify and rerun after each upstream synchronization and data repair before submission; Cargo may reuse valid build artifacts, but a previous pass is not a current validation result.

Review the entire batch diff against the freshly fetched tip with `git diff --name-status UPSTREAM_TIP...HEAD`, plus staged, unstaged, and untracked changes before committing. Allow only `meet-data/ps/` changes and the specific necessary, evidence-backed PS identity registry edits under `lifter-data/`. Review registry edits row by row for effects on other lifters. Stop on unrelated paths, unexplained deletions, private information, or unsupported edits. Keep workflow documents, local inputs/checkpoints, and generated build artifacts out of the data submission.

Commit only explicitly reviewed paths after reconciliation, checks, and scope review pass. Record the commit and validation. If new upstream changes are merged later, repeat reconciliation as needed, checks, and scope review before submitting.

## 5. Submit one batch MR

Repeat the synchronization gate before submission, then rerun checks if the branch changed. Confirm the push remote is the existing GitLab fork and the MR target is the original GitLab project and its discovered default branch. Push only the batch branch with a normal, non-force push. If rejected, inspect remote work and reconcile safely; never overwrite it.

Search again for an existing MR for this batch and reuse it. Otherwise create one MR for all ready meets in the batch, listing the included meets, public source evidence, justified identity changes, and validation results. Leave incomplete meets out and record them locally for a later batch. Record the MR URL and exact pushed head SHA; confirm the MR references that head before monitoring. Authentication, fork relationships, or target permissions that block submission require user assistance, not an alternate target.

## 6. Monitor and repair

Inspect the MR's current head and its required pipelines/jobs, including required downstream jobs. Tie each result to the **latest MR head commit**; for merged-results pipelines, verify the pipeline's source head matches that commit. A successful pipeline for an older head is insufficient. Record pipeline identity, commit association, and required job statuses. Pending or running jobs require continued monitoring using the available wait/monitor facility; missing, canceled, skipped, blocked/manual, or unreadable required checks are not proof of success.

For a failing job, inspect its logs and diagnose the cause. Begin each repair cycle with step 1, then make only source-verified, in-scope corrections, rerun the relevant and full required checks, review scope, commit, and submit to the same branch/MR through step 5. Monitor the new head again. Escalate infrastructure, permission, ambiguous-data, and out-of-scope failures with the evidence; do not weaken validation or alter unrelated data/CI to obtain green jobs.

## 7. Completion

Finish only when the MR still points to the recorded latest head and every required job for that head has passed. Recheck head and pipeline status immediately before reporting completion. Report the MR URL, head commit, local validation and current pipeline results, included meets, and any unresolved decisions. Include a dedicated "Missing/incomplete meets" section in every final handoff, including blocked or no-ready-meet runs: list each held meet, exact missing/unusable files or coverage, and the required next action. If there are no gaps, state that none were found within the inspected scope; do not imply uninspected meets were checked. Leave merging to upstream maintainers. If blocked, report the exact blocker and checkpoint needed to resume; do not label the batch complete.
