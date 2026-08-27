# Complete Implementation Plan — Encoding Failure Lab

This file is the executable backlog. Work in dependency order. Check boxes only after implementation + tests + build pass.

## Phase 0 — Bootstrap and project hygiene

### E0.1 Browser TypeScript project

Create:

```text
package.json
package-lock.json
tsconfig.json
vite.config.ts
src/
tests/
fixtures/
research/
```

Requirements:

- TypeScript strict mode;
- Vite production build;
- Vitest test runner;
- scripts for `test`, `build`, and `dev`;
- no backend;
- deterministic build output suitable for GitHub Project Pages.

Acceptance:

- [ ] clean install succeeds;
- [ ] empty/minimal test suite runs;
- [ ] production build succeeds.

### E0.2 CI

Add GitHub Actions that on push/PR:

- install from lockfile;
- run tests;
- run TypeScript checking if separate;
- build production site.

Acceptance:

- [ ] CI is green on main.

---

## Phase 1 — Transformation model

### E1.1 Domain types

Create `src/core/types.ts`.

Define explicit tagged representations for:

- Unicode text;
- byte sequences;
- operation direction;
- encoding label;
- error mode;
- warnings/assumptions;
- loss level L0–L4.

Do not allow an encode step to accidentally consume bytes or a decode step to consume text.

Acceptance:

- [ ] invalid domain transitions are rejected by types or runtime validation;
- [ ] tests cover construction/validation.

### E1.2 Stable representations

Create helpers for:

- hex bytes;
- code point list (`U+XXXX` / extended form);
- escaped text display;
- byte length and code point count;
- optional compact human label.

Acceptance:

- [ ] non-BMP characters and combining sequences display correctly;
- [ ] zero bytes and control characters remain inspectable.

### E1.3 Trace model

Create `src/core/trace.ts`.

A trace is an ordered sequence of immutable transformation steps. Each step should include:

```text
id
operation
input domain + stable representation
output domain + stable representation
encoding/transform label
error mode
loss level
loss reason
warnings/assumptions
```

Provide deterministic JSON serialization for golden tests and replay.

Acceptance:

- [ ] same input/history produces byte-for-byte stable JSON excluding explicitly documented nondeterministic metadata;
- [ ] trace can be replayed from stored step inputs where the adapter is available.

### E1.4 Loss classifier

Create `src/core/loss.ts`.

Encode repository rules for L0–L4. Keep policy separate from UI strings.

Required cases:

- exact encode/decode round trip → L0;
- reversible mojibake only with historical hypothesis → L1;
- multiple plausible originals/histories → L2;
- replacement character/substitution collapse → L3;
- deletion/overwrite/irreversible identity destruction → L4.

Acceptance:

- [ ] unit tests for every transition rule;
- [ ] loss reason is explicit, not inferred by the view.

### E1.5 Model documentation

Write `docs/MODEL.md` from the actual implemented model, not aspirational pseudocode.

Include at least 10 boundary examples and explain byte/text domains.

Acceptance:

- [ ] examples correspond to executable tests/fixtures where practical.

---

## Phase 2 — Encoding adapter layer

### E2.1 Browser capability survey

Write `research/browser-encoding-capabilities.md`.

Verify current behavior for:

- `TextEncoder`;
- `TextDecoder`;
- fatal/replacement handling;
- WHATWG legacy labels;
- GBK/GB18030;
- Big5;
- Shift_JIS;
- EUC-JP if supported;
- ISO-2022-JP;
- Windows-1252 and ISO-8859-1 label compatibility behavior.

Record browser/runtime tested and date.

Acceptance:

- [ ] no manually invented compatibility table;
- [ ] implementation decision for legacy *encoding* (text → bytes) is documented.

### E2.2 Adapter interface

Create `src/encodings/adapter.ts`.

Provide a uniform interface that keeps platform/library details outside core trace logic.

Acceptance:

- [ ] UTF-8 adapter works;
- [ ] one legacy single-byte adapter works;
- [ ] adapter errors map to explicit fatal/replacement outcomes.

### E2.3 Mature legacy encoding implementation

Use browser APIs and/or a mature dependency for missing text→bytes legacy encodes. Keep dependency isolated under `src/encodings/`.

Acceptance:

- [ ] no large project-maintained mapping table;
- [ ] representative vectors match authoritative implementation behavior.

---

## Phase 3 — Golden vectors and proof fixtures

### E3.1 Fixture schema

Create a documented fixture format under `fixtures/` containing:

- source text and/or raw source bytes;
- operation chain;
- expected intermediate representations;
- expected loss level;
- citation/note when historically or standards relevant.

### E3.2 Classic mojibake vectors

Include at minimum:

- UTF-8 Chinese text decoded under a compatible wrong single-byte path;
- Western punctuation that exposes Windows-1252-specific behavior;
- an example whose reverse hypothesis restores the exact original;
- an example that looks plausible but is not proof of original intent.

Acceptance:

- [ ] golden trace tests pass.

### E3.3 U+FFFD irreversible-loss proof

Create at least two distinct invalid byte inputs that collapse to insufficiently distinguishable replacement-decoded text, proving original byte identity cannot be recovered from the text alone.

Write tests and a short explanatory fixture note.

Acceptance:

- [ ] test demonstrates different original byte sequences;
- [ ] current Unicode output alone cannot uniquely identify which source was used;
- [ ] trace marks substitution loss.

### E3.4 Double-mojibake vectors

Create at least three chains:

- exactly recoverable with the correct history;
- recoverable only after two inverse hypotheses;
- not uniquely recoverable after a lossy step.

Acceptance:

- [ ] replay and branch comparison tests pass.

---

## Phase 4 — Core branch/replay engine

### E4.1 Branch from any valid state

Implement branch creation from a byte or text state so the same source can be interpreted with different next operations.

### E4.2 Compare branches

Create comparison helpers exposing:

- identical/different bytes;
- identical/different code points;
- loss level;
- exact recovery status;
- assumptions required.

### E4.3 Recovery claims

Implement explicit claim labels such as:

- exact inverse verified;
- matches a supplied hypothesis;
- plausible but ambiguous;
- impossible from current information.

Never equate “looks readable” with verified recovery.

Acceptance:

- [ ] branch/replay operates independently of UI;
- [ ] tests include same bytes under multiple decoders.

---

## Phase 5 — Exhibit 1: Classic Mojibake

Create `src/exhibits/classic-mojibake/` and corresponding web route.

Required interaction:

```text
original text
→ code points
→ UTF-8 bytes
→ choose wrong decoder
→ mojibake text
→ attempt inverse hypothesis
→ compare with original
```

Required display:

- text;
- code points;
- hex bytes;
- operation;
- encoding label;
- loss level;
- explanation of why this example remains recoverable.

Acceptance:

- [ ] default scenario works without typing input;
- [ ] user can change the wrong decoder where supported;
- [ ] exact recovery is verified by bytes/code points, not visual equality alone.

---

## Phase 6 — Exhibit 2: Point of No Return

Create `src/exhibits/replacement-loss/`.

Interaction:

- choose among several raw invalid byte sequences;
- replacement-decode them;
- inspect U+FFFD placement;
- compare two histories that have lost source identity;
- attempt recovery and receive an explicit “insufficient information” explanation.

Acceptance:

- [ ] at least two source byte sequences demonstrate collapse;
- [ ] fatal mode comparison is available;
- [ ] the exhibit makes clear that U+FFFD is a Unicode replacement character introduced by error handling.

---

## Phase 7 — Exhibit 3: Double Mojibake Timeline

Create `src/exhibits/double-mojibake/`.

Features:

- timeline of text/bytes states;
- step forward/backward through recorded history;
- branch from a selected step;
- manually change an encoding hypothesis;
- compare two candidate recovery paths;
- label exact recovery versus merely readable output.

Acceptance:

- [ ] entire timeline can be reconstructed from trace data;
- [ ] no correctness state exists only in UI component state.

---

## Phase 8 — Exhibit 4: CJK Failure Gallery

### E8.1 Research vectors

Create `research/cjk-failure-vectors.md` using authoritative sources/runtime verification.

Select examples for:

- GBK/GB18030;
- Big5;
- Shift_JIS;
- EUC-JP if useful;
- bytes valid in multiple encodings but yielding different text;
- multibyte boundary failures;
- valid-but-wrong characters versus replacement.

### E8.2 Gallery UI

Create `src/exhibits/cjk/`.

The exhibit should favor a curated gallery over an arbitrary giant converter matrix.

Acceptance:

- [ ] every case includes raw bytes;
- [ ] every case explains why the wrong result may still look like legitimate CJK text;
- [ ] detector/probability language is not presented as certainty.

---

## Phase 9 — Exhibit 5: Stateful Encoding

Create research note and exhibit for ISO-2022-JP or another justified stateful WHATWG encoding.

Display decoder state transitions explicitly.

Acceptance:

- [ ] user can see that interpretation depends on prior escape/state changes;
- [ ] implementation uses verified decoder semantics rather than a made-up state machine unless explicitly labeled as a pedagogical simplification.

---

## Phase 10 — Adjacent transformation layers

Only after core encoding exhibits are stable, add a separate section for adjacent failures.

Candidate modules:

- BOM/endian interpretation;
- Unicode normalization;
- HTML entity escaping;
- JSON escaping;
- URL percent encoding;
- Base64 as a transport representation;
- filesystem filename conversion.

Rules:

- every module must label its layer correctly;
- do not add a module solely because it produces ugly text;
- show where bytes/text boundaries occur between layers.

Acceptance:

- [ ] encoding and non-encoding transformations are visually and structurally distinguishable.

---

## Phase 11 — Optional comparison tools

### E11.1 ftfy comparison

If practical, add a read-only comparison explaining what a mature repair heuristic proposes versus what the recorded history proves.

Do not copy ftfy algorithms.

### E11.2 detector comparison

Optional exhibit showing that a detector returns candidates/confidence, not recovered metadata.

Do not turn the repository into a detector benchmark.

---

## Phase 12 — Public exhibit shell

Build a coherent static site with routes approximately:

```text
/
/classic-mojibake
/replacement-loss
/double-mojibake
/cjk
/stateful
/adjacent
/about
```

Site-wide components:

- transformation timeline;
- byte inspector;
- code point inspector;
- loss badge with explanation;
- branch comparison;
- source/method note;
- “show details/math/standard note” progressive disclosure.

Accessibility:

- keyboard operable;
- sufficient textual explanation without animation;
- reduced-motion friendly;
- no meaning encoded only by color.

Acceptance:

- [ ] a visitor can understand the first two exhibits on mobile/desktop without cloning the repo.

---

## Phase 13 — GitHub Pages deployment

Add deployment workflow for Project Pages.

Requirements:

- correct Vite base path for repository Pages;
- test/build gate before deployment;
- no dependency on `tmzncty.github.io`;
- no secret server component.

Acceptance:

- [ ] public Project Pages URL loads all routes/assets correctly;
- [ ] direct navigation/reload behavior is handled appropriately for static hosting.

---

## Phase 14 — Documentation and educational finishing

Update README to include:

- live demo link;
- screenshot/GIF only if it helps, not as correctness evidence;
- quick explanation of loss taxonomy;
- links to model, research, and implementation docs.

Add `docs/TEACHING_NOTES.md` explaining a suggested order:

1. bytes vs text;
2. classic recoverable mojibake;
3. substitution boundary;
4. double histories;
5. CJK ambiguity;
6. stateful decoders;
7. adjacent layers.

Acceptance:

- [ ] claims about standards/current browser behavior are sourced or linked to research notes;
- [ ] no historical or compatibility simplification is stated as universal fact.

---

## Phase 15 — Final verification matrix

Before calling the project complete, run and record a matrix covering:

- supported desktop Chromium-family browser;
- Firefox;
- WebKit/Safari-equivalent if available in CI/manual testing;
- production Pages build;
- all golden fixtures;
- fatal/replacement paths;
- all exhibit default scenarios;
- keyboard navigation smoke test.

Create `docs/VERIFICATION.md` with date, commands, and known limitations.

---

# Cross-cutting quality requirements

## Determinism

Core histories and fixtures must serialize deterministically.

## Provenance

A visitor should be able to determine which bytes existed before a lossy step when those bytes are known by the experiment.

## No false certainty

Readable output is not proof of original intent. Detection is not recovered metadata. A guessed inverse chain is a hypothesis until verified against known source identity.

## No mapping-table hobby project

The project owns trace semantics, loss reasoning, branching/replay, and visualization. It does not own Unicode/legacy character tables.

## Keep it playful

This is a lab/exhibit, not an enterprise conversion product. Curated failure specimens, branching experiments, and visible “point of no return” moments are preferred over giant forms and settings panels.

# Completion criterion

The required project is complete when Phases 0–9 and 12–15 are done and deployed. Phases 10–11 are valuable extensions but must not delay a polished core release unless implementation naturally reaches them.

After the core release, agents may continue with adjacent layers only when each new module adds a distinct explanatory failure mechanism.