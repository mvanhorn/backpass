# Candidate selection (internal)

Not for the public plan files. Public docs must not name other projects or claim inspiration.

Date: 2026-09-27
Upstream audit: `kunchenguid/backpass` issues and PRs via `gh` (all issues, open PRs, closed feat PRs). Discussions disabled.

## VISION accept/resist used as the score

Align: stronger evidence, cheaper to verify, harder to fabricate; say no faster; say no once and have it remembered.

Resist: lower the two-session bar; widen writes; default anything but per-edit review; easier grow than shrink; API keys/servers/others' transcripts; coverage over accuracy; quieter degraded paths.

## Upstream already taken (do not propose)

| Item | Where |
| --- | --- |
| Team mode | #24 |
| Plugin / producer-repo scope | #117 |
| Multiple memory files as one always-loaded surface | #111 |
| Nested AGENTS.md in monorepos | #158, PR #159 |
| Copilot CLI harness | #25 |
| OMP discovery | #53, PRs #129/#150/#151 |
| Steering / task instructions as citable sources | PR #154 |
| SSH multi-machine discovery | merged #124 |
| `--target` | merged #108 |
| User scope | merged #101 |
| skillSearchPaths | merged #138 |
| Apply funnel / cross-surface overlap (report-only) | merged #104, #81, #69 |

Closed-not-planned adjacent: #34 (instruction-level evidence cache invalidation). Coarse `memoryHash` invalidation stays; accuracy over cheaper reruns. A "file as of session time" analyzer would be a different feature, but it sits next to that grain and was not chosen.

Not features: open bugs (#148, #147, #127, #121, #116, #115, #112, #91, #76, #75, #70, #57, #50, #35, #23). Plans must not be fixes for those.

Closed without merge that look like features we should not revive: cwdAliases (#77), Antigravity adapter (#56), standalone OMP adapter (#21).

## Candidates weighed

| Candidate | User value | VISION | Size | Risk | Verdict |
| --- | --- | --- | --- | --- | --- |
| Fold-anchored quotes (catalog gate + context window) | High: every apply | Harder to fabricate, cheaper to verify | Medium | Low-medium (ambiguous substring, fold 6-cap) | **Pick 1** |
| Reword-stable rejection identity + optional reasons | High: repeat review | Say no once; say no faster | Medium | Medium (false suppress) | **Pick 2** |
| Contemporaneous git snapshot of the memory file at session start | High accuracy in theory | Stronger evidence, but adjacent to #34; git often missing; CLAUDE.md uncommitted | Large | High fail-soft / false non-compliance | Defer |
| Instruction-coverage / dead-weight `status` report | Medium | Presentation never gates; skipped rules argue for reinforce not delete; easy to read as "delete uncited lines" | Small-medium | Misaligned deletions | Reject |
| Apply-time re-distill of raw transcripts | Medium | Accuracy, but couples apply to discovery/SSH cache | Medium | Remote/missing files fail loud or degrade | Folded into pick 1 as *not* doing this |
| New harness adapters (beyond Copilot/OMP) | Coverage | Welcome in VISION, not the evidence/reject bar | Medium each | Adapter drift | Not this pair |
| Semantic rejection via bigram of added lines | Looks like say-no-once | Forbidden lexical/text-shape classifier on a gate | Small | False suppress | Reject |
| Auto-apply / approve-all default | Speed | Resist: per-edit review is the default | Small | Out | Reject |
| Lower `minGapEvidence` for "high confidence" quotes | More edits | Resist the two-session bar | Small | Out | Reject |

## Why these two beat the rest

Analysis already proves quotes against the distilled trace. Synthesis annotation and apply display do not. That is an unenforced VISION sentence ("nothing the model says is taken on faith") with no GitHub issue. Anchoring annotate quotes to the fold catalog, and showing the mechanical window at apply, is net-new and meaty without touching adapters or floors.

Rejection memory is also a VISION sentence whose implementation is hunk-hash only. `test/proposal.test.js` even documents that a different destination is a different edit, which is correct, but a reworded add of the same gap is *not* a different decision. Using ledger `entry.id` and measured instruction units reuses identity the project already trusts (consolidation, `unitsRemovedBy`). Named reasons are optional metadata, not a floor.

Contemporaneous snapshots would also strengthen evidence, but #34 closed not-planned on keeping judgments tied to the current surface hash. Teaching the model "this line was added after the session" as display is possible later; using old judgments as corroboration would fight the closed design question.

## Other tools (public plans must not mention these)

Several local tools mine coding-agent transcripts into markdown memory: session-end compilers that append daily logs and concept articles; Cursor-hook updaters that write learned preferences into `AGENTS.md`; Claude JSONL miners that emit NDJSON insights; transcript "doctors" that generate paste-ready rules from anti-pattern scores.

They optimize for capture and growth. Backpass already refuses that shape: two-session corroboration, harm-only deletion, per-edit review, remembered rejection, no product-owned API key. The two plans deepen *verification and refusal* on that existing loop. They do not add an append-only compiler, a hook that writes instruction files, or a rule generator that bypasses `buildProposal`.

## Sequencing

If both are implemented, land fold-anchored quotes first. Reword-stable `gapIds` should share the unique-substring catalog join; implementing a weaker exact-string join first would have to be ripped out.

## Slack / Granola

No backpass-maintainer meeting notes were used as requirements. Granola hits were about other agent-memory products and were treated as non-authoritative. Slack was not searched (not requested).
