# Fold-anchored quotes

Date: 2026-09-27

A proposal quote is only as good as a reviewer's ability to check it. Analysis already discards paraphrases that do not appear in the distilled trace. Synthesis annotation does not: it may attach any text as long as the source label was issued by this run's fold. Apply then shows a 200-character snippet and a harness id, with no surrounding moment. This plan closes that hole so a person can verify a quote in one sitting, and so a fabricated quote cannot clear the gates.

## Problem

`sanitizeEvidence` in `src/analyze.js` requires a verbatim quote and, unless `usedRawTranscript === true`, a whitespace-folded substring of the distilled trace. That is the analysis-stage rule.

The annotate turn is a different model call. `normalizeEvidence` in `src/proposal.js` keeps polarity, a 600-character `text`, and a `source` string. `countSources` then counts a quote as a session only when `normalizeSourceLabel` matches a fold-issued label from `summary.sources`. The quote body is never checked against the fold catalog or the trace. A test in `test/proposal.test.js` pins the source-label rule ("a quote counts as a session only when the fold issued its source label") and does not pin quote-body membership.

The reviewer pays for that. `renderEdit` in `src/apply/terminal.js` prints the first four quotes at 200 characters. `templates/apply.html` renders quote text plus `.src`. There is no locator, no surrounding distilled moment, and no badge that the quote was mechanically found. Checking one card means hunting the local session store by native id.

That is the opposite of cheaper to verify, and it is a path for fabricated evidence: a real source label plus invented text still counts toward `minGapEvidence`.

## User-facing behavior

After a run, `backpass apply` (browser and `--no-ui`) shows each evidence quote with a mechanically extracted window of the distilled moment it came from: a short before-span, the quote itself, and a short after-span, plus the existing source label (`harness · nativeId · date`, with host when collected over SSH).

A quote that analysis never recorded for that source does not appear on a shippable edit. If synthesis annotates with text that is not a whitespace-folded substring of a fold-issued quote from that same source, `buildProposal` records a named violation and the annotate re-prompt path in `src/synthesize.js` asks for a real catalog quote. The run still fails loudly after `ANNOTATE_TURNS` rather than shipping the invented text.

Older evidence files that predate locators still count as sessions when the quote text matches the catalog. Apply omits the expandable window in that case and does not pretend context exists. Quotes accepted only because the analysis model set `usedRawTranscript` likewise have no distilled-trace locator; they remain catalog-gated on text and source, with no window until a later pass that can locate them in the distilled trace.

Nothing here lowers `minGapEvidence`, changes deletion harm rules, writes the repo outside `src/apply/writer.js`, or auto-accepts an edit.

## Design

### Catalog, not a second model

Fold already builds the only quote catalog a later stage should trust. `foldEvidence` in `src/fold.js` attaches instruction-row quotes (`text`, `source`, `polarity`, `class`, `effect`, `moment`) and gap-cluster quotes (`text`, `source`, `effect`). Instruction rows and eligible gap items keep at most six quotes each (`quotes: entry.quotes.slice(0, 6)` and `eligibleItems.slice(0, 6)`). `renderEvidenceForPrompt` then shows three of those to synthesis. The gate uses the fold-emitted catalog (the six), not the prompt slice and not leftover quotes sitting only on raw evidence files. A seventh analysis quote that fold dropped cannot satisfy annotation; that is the existing fold cap, documented rather than raised.

### Locate after a successful trace check

Add `locateQuote(trace, quote)` next to `foldSpace` in `src/analyze.js`. After `sanitizeEvidence` accepts an item against `distilled.trace`, compute:

- `foldedOffset`: index of the folded quote in the folded trace
- `before` / `after`: up to 120 characters of the distilled trace on each side of the match, not model-authored
- optional `turn` / `role` when the match sits under a `### turn N · role` heading from `src/distill.js`

Attach `{ locator, before, after }` on the stored positive, negative, and gap items. Do not locate quotes that passed only via `usedRawTranscript`. Do not persist the full trace. Do not bump `ANALYSIS_INDEX_VERSION`: locators are mechanical post-processing, not a change to what analysis accepts from the model. Cached evidence without locators stays fresh.

`foldEvidence` copies locator fields onto instruction-row and gap quotes when present. Gap quotes that arrive only through `ledgerGapObservations` often lack locators; that is the same fail-soft as old evidence, not a ledger schema change in this work.

### Annotate quotes must be catalog substrings

In `src/proposal.js`, before `countSources` runs, resolve each annotate evidence item against the fold catalog:

1. Restrict candidates to quotes whose `normalizeSourceLabel` equals the item's source (and the source is in `summary.sources`).
2. Keep candidates where `foldSpace(item.text)` is a substring of `foldSpace(candidate.text)` and the item text is at least 8 characters (the analysis minimum).
3. If zero matches: violation, named so the annotate re-prompt can quote the rule. If two or more catalog quotes from that source match: violation (ambiguous substring). First-match overwrite is forbidden, because one session can carry a positive and a negative that share a short span.
4. On a unique match, replace polarity, class, locator, and context from the catalog. Keep the model's shorter `text` when it is a true substring, so the card can highlight the cited span inside the window. Drop `neutral` unless the catalog itself carried it.

`countedEvidenceProjects` currently joins gap quotes with an exact `` `${source}\n${text}` `` set. That join must use the same unique-substring matcher, or a shortened catalog quote would silently lose user-scope project credit and fail `minGapProjects` for the wrong reason.

Session counts, token deltas, and budget numbers remain measured in `buildProposal`. The catalog match does not introduce a lexical classifier of instruction meaning; it is membership in a list fold already emitted.

### Apply surfaces

`renderEdit` in `src/apply/terminal.js` prints the before/quote/after window when present, still capping the quote line, and still listing source underneath.

`templates/apply.html` keeps the existing `<details class="evidence">` list. Each quote that has `before`/`after` becomes expandable context; polarity color stays on the quote. The injected payload is still the proposal JSON via `injectPayload` in `src/apply/lavish.js` (function replacer, never a string replacer).

Apply does not re-distill transcripts and does not spawn discovery. Context is already on the proposal.

### What this does not change

Adapters, golden fixtures, `src/distill.js` clamping, gap-ledger identity, `minGapEvidence`, harm vs non-compliance, and the writer. Presentation still does not gate: a missing window is not a reason to refuse an otherwise valid edit.

## Assumptions

- Unique-match failures are treated as gate violations, not as "pick the first quote."
- The fold six-quote cap stays; raising it is separate work if annotate re-prompts cite dropped quotes in real corpora.
- `usedRawTranscript` quotes stay legal without locators so the raw-transcript escape hatch is not punished.

## Test plan

No adapter golden fixtures change. Offline tests only; `pnpm run check` is the bar.

`test/proposal.test.js`

- An annotate quote whose source is fold-issued but whose text is not a substring of any catalog quote from that source is a violation, even when two such invented quotes would have met `minGapEvidence` under today's source-only count.
- A unique substring of a catalog quote inherits catalog polarity, class, and locator; `transcripts` still counts the fold-issued source once.
- Two catalog quotes from the same source that both contain the substring is a violation, not first-match.
- `countedEvidenceProjects` credits a gap cluster when the annotate text is a unique substring of that cluster's quote, not only on exact string equality.
- A source label the fold did not issue still does not count (existing test remains true).

`test/fold.test.js` / analyze unit tests beside `sanitizeEvidence`

- After a successful trace match, the stored item has `before`/`after` sliced from the distilled string, not from the model.
- `usedRawTranscript` items have no locator.
- `foldEvidence` copies locator fields onto instruction-row quotes and onto in-run gap quotes; ledger-only observations without locators still cluster.

`test/apply-cli.test.js` / `test/apply-surface.test.js`

- Terminal `renderEdit` includes the window when present and omits a fake window when absent.
- `injectPayload` HTML contains the quote and the before/after text for a fixture proposal. Fake `lavish-axi` polling is unchanged.

Fake analysis agents must keep quoting real fixture session text. Do not bump `ANALYSIS_INDEX_VERSION` unless sanitize's accept rule changes during implementation.

## Rough size

Medium. The work is concentrated in `src/analyze.js`, `src/fold.js`, `src/proposal.js`, `src/apply/terminal.js`, and `templates/apply.html`, with tests in the files above. No new runtime dependency, no new subcommand, no adapter, no SSH change. The annotate violation strings must stay specific enough for the existing re-prompt loop.

## Demo idea

A short terminal recording of `backpass apply --no-ui` on a fixture proposal: first card shows a negative quote, then three lines of distilled context around it, then the source label. Cut to a second recording (or a saved HTML apply page) where synthesis tried to attach a paraphrase with a real source label: stderr prints the named catalog violation, no `proposal.json` edits ship, and the apply surface is never offered. The HTML demo is the same card with the evidence `<details>` expanded so the before/quote/after window is visible without scrolling the whole funnel.
