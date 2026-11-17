# Powerlifting Singapore batch workflow

Use this procedure when asked to process or submit PS results, resume a batch, or monitor and repair its CI. Creating or editing this setup does not invoke processing or submission.

## Scope and local state

This GitHub repository stores the workflow. Inputs live in ignored `data/`; checkpoints and intermediate exports live in ignored `.workflow-state/`. The sibling `../open-powerlifting-database-maintenance` checkout holds the data branch. Submit that branch through its existing GitLab fork, `hud.syafiq/open-powerlifting-database-maintenance`, to the original `openpowerlifting/opl-data` project on GitLab. Never submit data through a GitHub PR or push directly to the original project.

Keep a batch checkpoint in `.workflow-state/` with:

- Input filenames and fingerprints, meet/date/day/platform coverage, evidence and export paths, destination meet directories, and ready/held/submitted status.
- Data checkout, branch, freshly fetched upstream default branch and tip, fork and target project, MR URL, and submitted head commit.
- Validation commands/results, pipeline ID/URL and associated commit, required job results, and unresolved decisions with their evidence.

Update the checkpoint after each completed step and before handoff. On resume, inspect the actual branch, files, MR, and pipelines; reconcile them with the checkpoint instead of trusting cached status. Keep credentials in the existing credential manager or environment, outside tracked files and checkpoints. Preserve source files. This workflow installs no global skills, hooks, scheduled jobs, parent-folder instructions, or global configuration.

## 1. Synchronize upstream — mandatory gate

Run this step first on **every invocation, including resumed and CI-monitoring runs**, and repeat it before each repair or submission cycle. A saved upstream tip or a clean CI result never replaces a fresh synchronization.

1. In the sibling checkout, inspect `git status --short --branch`, the current branch, local batch branches, and remotes. Identify any existing batch and submission before creating another. If tracked changes, relevant untracked work, an unfinished Git operation, or conflicts prevent safe synchronization, stop and ask the user to resolve them. Preserve all work; do not automatically stash, discard changes, reset, or force-push.
2. Discover the original repository's current default branch with `git ls-remote --symref https://gitlab.com/openpowerlifting/opl-data.git HEAD`. Fetch that advertised branch directly from the original URL with `git fetch https://gitlab.com/openpowerlifting/opl-data.git refs/heads/BRANCH`, replacing `BRANCH` with the discovered name. Immediately record `git rev-parse FETCH_HEAD` as the upstream tip. A fetch from the fork alone is insufficient. If discovery or fetch fails, stop; do not use stale refs.
3. For a new batch, create a uniquely named `ps/<batch>` branch from that exact fetched tip. For an existing batch, switch to its recorded branch and merge that fetched tip into it, preserving batch commits. If the branch exists only on the fork, fetch and resume it rather than starting a replacement. Stop on conflicts and request resolution; leave the work intact.
4. Require `git merge-base --is-ancestor UPSTREAM_TIP HEAD` to succeed, using the freshly recorded SHA. Record the successful gate in the checkpoint. Until it succeeds, do not convert, edit, repair, validate for submission, or push data. Repeat validation after any upstream merge that changes the branch.

## 2. Inventory inputs and prevent duplicates

List every file in `data/` and map it to a meet, date, day, and platform in the checkpoint. Inspect actual source contents, not just filenames. Compare against the freshly synchronized `meet-data/ps`, existing fork branches, and open or merged upstream submissions, including the batch MR. Match meet metadata and lifter/results coverage; different filenames or meet IDs do not prove distinct meets. Reuse an existing submission for the same batch. Skip already submitted results; treat corrections as evidence-backed updates to their existing meet rather than duplicate meets. If submission history cannot be checked, stop before creating a potentially duplicate submission.

Confirm that each meet has final results for every expected day and platform. Hold incomplete meets and record precisely what is missing; continue only with independently complete meets. In particular, a single day/platform backup does not establish that a multi-day or multi-platform meet is complete. If nothing is ready, report the held inputs and stop without an MR.

For `.openlifter` backups, load a copy through OpenLifter and export its OpenPowerlifting CSV. Keep the backup and export mapping locally, verifying all days/platforms against the backup and other source evidence. If OpenLifter is unavailable or the export cannot be verified, hold that input and request assistance; do not invent a backup parser or fabricate missing results.

## 3. Process and reconcile

Read the fetched upstream documentation, nearby PS meets, and relevant tool usage before conversion. Follow the PS layout: one `meet-data/ps/YYNN/` directory per meet, containing:

- `meet.csv`: meet metadata, matching the existing PS schema.
- `original.csv`: the unchanged source CSV, retained beside the processed data. For multiple source sessions, preserve each original with a distinct day/platform/session suffix, following PS precedents such as `original-D1AM.csv`.
- `entries.csv`: the final processed results covering the complete meet, reconciled with all source CSVs.

`YY` is the meet year's last two digits; `NN` is the zero-padded meet sequence for that year, starting at `01` (for example, `2601` is the first meet of 2026, not January). Allocate new meets in chronological meet-date order, not input filename, processing, or submission order. Check existing meets and pending submissions before assigning numbers and record the mapping in the checkpoint. Preserve published meet IDs, including historical ordering exceptions; if fitting a new meet chronologically would require renumbering existing IDs, or the ordering is ambiguous, ask the user before assigning it.

Reuse upstream tools only for compatible source formats: `scripts/openlifter-convert` consumes a full OpenLifter `original.csv` export in the current meet directory and produces `meet.csv` and `entries.csv`. Run it only in the intended new meet directory after copying the verified export there as source evidence. For other formats, follow the corresponding existing upstream conversion pattern or request clarification. Keep private backups and working artifacts local; include only source evidence appropriate for public submission under upstream conventions.

Reconcile converted metadata, entry counts, identities, divisions, equipment, bodyweights, attempts, totals, and placings against the sources, accounting for all days/platforms and any legitimate repeated entries. Retain an evidence trail for corrections. Ask the user about uncertain results or identities; hold affected meets until resolved rather than guessing or deleting troublesome entries. Change a shared identity registry only when necessary for these PS entries and supported by evidence; never overwrite another lifter's identity to silence a checker error.

## 4. Validate data and enforce scope

For every ready meet, verify that its folder contains the metadata, original source CSV(s), and final entries; that the folder year matches the meet date; and that new meet numbers follow the agreed chronological mapping without collisions.

In the data checkout, run the fetched upstream checks. Use its current `Makefile`, `tests/`, and `.gitlab-ci.yml` as the source of truth: currently `make check` runs the data and lifter checks, with `tests/check-line-endings` and `tests/check-file-sizes` providing additional CI checks. Run any other required checks selected by the changed files. Record commands and results. A missing dependency, failed check, or unavailable check is a blocker to claiming validation, not a pass.

Review the entire batch diff against the freshly fetched tip with `git diff --name-status UPSTREAM_TIP...HEAD`, plus staged, unstaged, and untracked changes before committing. Allow only `meet-data/ps/` changes and the specific necessary, evidence-backed PS identity registry edits under `lifter-data/`. Review registry edits row by row for effects on other lifters. Stop on unrelated paths, unexplained deletions, private information, or unsupported edits. Keep workflow documents, local inputs/checkpoints, and generated build artifacts out of the data submission.

Commit only explicitly reviewed paths after reconciliation, checks, and scope review pass. Record the commit and validation. If new upstream changes are merged later, repeat reconciliation as needed, checks, and scope review before submitting.

## 5. Submit one batch MR

Repeat the synchronization gate before submission, then rerun checks if the branch changed. Confirm the push remote is the existing GitLab fork and the MR target is the original GitLab project and its discovered default branch. Push only the batch branch with a normal, non-force push. If rejected, inspect remote work and reconcile safely; never overwrite it.

Search again for an existing MR for this batch and reuse it. Otherwise create one MR for all ready meets in the batch, listing the included meets, public source evidence, justified identity changes, and validation results. Leave incomplete meets out and record them locally for a later batch. Record the MR URL and exact pushed head SHA; confirm the MR references that head before monitoring. Authentication, fork relationships, or target permissions that block submission require user assistance, not an alternate target.

## 6. Monitor and repair

Inspect the MR's current head and its required pipelines/jobs, including required downstream jobs. Tie each result to the **latest MR head commit**; for merged-results pipelines, verify the pipeline's source head matches that commit. A successful pipeline for an older head is insufficient. Record pipeline identity, commit association, and required job statuses. Pending or running jobs require continued monitoring using the available wait/monitor facility; missing, canceled, skipped, blocked/manual, or unreadable required checks are not proof of success.

For a failing job, inspect its logs and diagnose the cause. Begin each repair cycle with step 1, then make only source-verified, in-scope corrections, rerun the relevant and full required checks, review scope, commit, and submit to the same branch/MR through step 5. Monitor the new head again. Escalate infrastructure, permission, ambiguous-data, and out-of-scope failures with the evidence; do not weaken validation or alter unrelated data/CI to obtain green jobs.

## 7. Completion

Finish only when the MR still points to the recorded latest head and every required job for that head has passed. Recheck head and pipeline status immediately before reporting completion. Report the MR URL, head commit, local validation and current pipeline results, included meets, and any held inputs or unresolved decisions. Leave merging to upstream maintainers. If blocked, report the exact blocker and checkpoint needed to resume; do not label the batch complete.
