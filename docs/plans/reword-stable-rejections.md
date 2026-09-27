# Reword-stable rejections

Date: 2026-09-27

A rejected edit is supposed to stay rejected until materially new evidence arrives. Today that memory is a hash of kind, file, and hunk body. Synthesis can change a few words, produce a different `find`/`replace`, and the same two sessions come back as a new card. This plan remembers the refused *gap* and the refused *instruction units*, still revives only on a higher measured session count, and optionally records why the person said no.

## Problem

`rejectionKey` in `src/state.js` hashes `kind`, `file`, and the hunk `find`/`replace` body. `isSuppressedByRejection` hides a later edit only when that key matches and `edit.transcripts` is not strictly greater than the stored count. `recordRejection` writes `kind`, `file`, `title`, `transcripts`, and `rejectedAt` into `.backpass/rejections.json` (or the user-scope state dir) after a successful decide path in `src/apply/writer.js`. Dry-run and any failed freshness, budget, or composition gate record nothing.

That is enough when synthesis proposes the same bytes. It is not enough when synthesis proposes the same decision in different bytes. `renderRejections` in `src/synthesize.js` lists titles and session counts for the prompt; the model is asked not to re-propose them. Prompt discipline is not a rule. The human spends review effort twice on the same answer.

Apply itself is only ACCEPT or REJECT. `reviewInTerminal` is `[a]ccept / [r]eject / [q]uit`. The browser card sets `decisions[edit.id]` to `accepted` or `rejected` and queues `BACKPASS_DECISIONS e1=accepted e2=rejected`. There is no place to say the quote was wrong, the rule already exists, or the wording is too narrow, so the next synthesis pass cannot see why the last one was refused.

This is not a lexical classifier on evidence floors. The two-session bar, harm-only deletion, and extract/move exemptions stay in `buildProposal`. Rejection identity is a separate suppress list the human already opted into by pressing reject.

## User-facing behavior

Rejecting an `add`, `rewrite`, or `remove` remembers more than the hunk bytes. A later proposal of the same kind against the same file is suppressed when it cites the same gap identities or touches the same measured instruction units, unless it is backed by strictly more fold-issued sessions than when it was turned down. `extract` and `move` keep today's hunk-key behavior only: repositioning or paying for a skill is not the same decision as adding or deleting instruction text.

`backpass apply` still decides one edit at a time. On the browser surface, REJECT may offer an optional reason chip: `wrong-evidence`, `already-covered`, `too-narrow`, `too-broad`, `disagree`. Skipping the chip still records the rejection. `--no-ui` records the rejection without asking for a reason, so a terminal review does not grow a second prompt. Unknown or malformed reason tokens are dropped; the accept/reject verdict still stands.

`backpass status` continues to report how many rejections are remembered. The next propose pass lists those identities (and reasons, when present) in the synthesis prompt. Suppressed edits are omitted from the apply surface, as they are today. Revival still means more measured sessions on that identity, never a reason code and never a model-reported count.

A run whose accepted subset fails a writer gate still records no rejections. Saying no is remembered only when the decide path actually completed.

## Design

### Stamp identities on the proposal, keep apply dumb

`recordRejection` today cannot see `summary` or `memoryFile`. Do not pass those into the writer. In `buildProposal`, after an edit is accepted (not suppressed, not violating), stamp two measured fields onto the edit object that `proposal.json` already serializes:

- `gapIds`: ledger entry ids for gap clusters whose catalog quotes uniquely match this edit's evidence under the same unique-substring rule as fold-anchored quotes. Clusters must carry a stable `id`. `ledgerGapObservations` in `src/gap-ledger.js` should include `gapId: entry.id` on each observation; `clusterGapObservations` / the decided cluster objects in `src/fold.js` should surface that id (union of item ids, or the surviving id after `mergeGapEntries`). Do not hash the cluster's current shortest phrasing with `gapEntryId` at reject time: that id would drift when the canonical sentence shortens.
- `instructionIds`: instruction units actually touched by the edit's memory-file hunks, using `unitsRemovedBy` and the same range intersection already used for removal evidence. Do not trust `edit.instructions` from the annotate JSON unless each id is in that measured set. Model-padded instruction lists must not widen suppression.

Writer then persists those arrays plus optional `reason` beside the existing hunk key. Entries missing the new fields (old `rejections.json`) keep hunk-key suppression only. Leave `version` at 1; the fields are additive.

### Suppression scan

`isSuppressedByRejection(edit, rejections)`:

1. If `rejectionKey(edit)` hits an entry and `transcripts <= prior.transcripts`, suppress (today's rule, all kinds).
2. If `edit.kind` is `add`, `rewrite`, or `remove`, also suppress when `kind` and `file` match an entry and the intersection of `gapIds` or of `instructionIds` is non-empty and `transcripts <= prior.transcripts`.
3. `extract` and `move` never take path 2.

Revival is still a strictly higher `edit.transcripts` measured by `countSources`. Two overlapping identities: suppress if *any* matching entry still outranks this edit's session count. Do not merge kinds: rejecting an `add` for a gap does not hide a later `remove` of a different instruction that happens to reuse a quote.

Cross-file extracts that already use distinct keys stay distinct. User-scope rejections stay in the user state directory; project runs never read them (`src/scope.js`).

### Optional reasons, never floors

Extend `parseDecisions` in `src/apply/lavish.js` so `e2=rejected:already-covered` is valid and `e2=rejected` remains valid. Keep the writer predicate `decisions[id] === "rejected"` for the verdict. Thread `rejectReasons` as a side map from `cmdApply` into `applyDecisions` / `recordRejection`. Invalid tokens are ignored.

`templates/apply.html` sets `decisions[id]` to `rejected` as today, and may set a parallel reason when a chip is chosen, encoding it on the APPLY vector. `window.lavish.queuePrompt` `data.decisions` can carry the same map; parsing stays tolerant because the vector still travels through a comment box.

`renderRejections` includes identity ids and reason so synthesis can avoid the refused wording without inventing a new classifier. Reasons do not change `minGapEvidence`, harm, extract/move exemptions, or revival.

### Sequencing with fold-anchored quotes

Quote-to-gap matching for `gapIds` should use the same unique-substring catalog join as fold-anchored quotes. If that work has not landed, this plan still ships hunk-key + measured `instructionIds` + reasons, and matches gaps only on exact `` source + text `` against `summary.gaps` quotes (today's `countedEvidenceProjects` join). Do not invent a second fuzzy matcher.

## Assumptions

- Terminal apply does not prompt for a reason in this work.
- Identity suppression is same-kind only; a rejected add can still be followed by an extract of existing text.
- Failed writer gates still record no rejections, including an all-reject session that fails a freshness check on a skill file the person did not accept.
- Suppression never uses bigram similarity of added lines. Gap identity stays the ledger id judged by consolidation; instruction identity stays measured unit ids.

## Test plan

No adapter golden fixtures. `pnpm run check`.

`test/proposal.test.js` (extend the existing revival cases around `isSuppressedByRejection`)

- Same hunk body, same transcript count: still suppressed.
- Same hunk body, more fold-issued sessions: still revived.
- Different hunk body, same `gapIds`, same kind and file, same or fewer sessions: suppressed.
- Same `gapIds`, strictly more sessions: revived.
- `extract` with a new body that shares a `gapId` with a rejected add: not suppressed by path 2 (hunk-key only).
- `rewrite` whose measured `instructionIds` overlap a rejected rewrite of those units: suppressed at equal session count.
- Annotate `instructions` that name units the hunks do not touch: those ids are not stored, and do not suppress an unrelated rewrite.
- A reason string is copied onto the rejection entry and does not appear in `violations`.

`test/user-apply.test.js` / writer tests

- Dry-run and budget-incompatible accepted subsets still leave `rejections.json` untouched.
- Successful reject-only apply writes `gapIds` / `instructionIds` from the proposal edit.

`test/apply-surface.test.js`

- `parseDecisions` accepts `e1=accepted e2=rejected:too-narrow` and ignores `e3=rejected:not-a-reason` as a reason while still recording rejected.
- Existing `e1=accepted e2=rejected` vectors still parse.

`test/synthesize.test.js`

- `renderRejections` includes a stored reason and a gap id so the prompt cannot silently drop them.

Do not add a text-shape classifier test that treats two different gap ids as one because their titles overlap.

## Rough size

Medium. State, `buildProposal`, fold/ledger id plumbing, apply HTML + `parseDecisions`, `renderRejections`, and the proposal/apply tests. No new dependency, no new subcommand. Risk is false suppression from over-wide `gapIds`; the mitigation is unique catalog matches and measured instruction ids only.

If fold-anchored quotes is in flight, land its catalog matcher first so `gapIds` and project counting share one join.

## Demo idea

Two terminal takes on the same fixture corpus. First: `backpass apply --no-ui`, reject an add that cites two sessions of a named gap, no write. Second: run propose again on the same evidence; stdout/TUI shows the previously rejected edit suppressed, and the apply surface has no card for a reworded add of that gap. Optionally expand the HTML card once: reject with `already-covered`, then show `backpass status` still listing one remembered rejection and the next synthesis prompt containing that reason. Revival is a third take with a third session's quote on the same gap: the card returns, labeled with three transcripts.
