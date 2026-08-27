# AGENTS.md — Encoding Failure Lab execution contract

This repository is designed to be continued by coding agents without repeated human planning.

## Mission

Build a public, static, testable interactive lab that makes encoding failures observable as **transformation histories** rather than presenting another converter or one-click mojibake fixer.

The project must answer:

- what object currently exists: bytes or Unicode text;
- which encode/decode operation happened;
- which encoding label and error mode were used;
- whether the step is exactly reversible, recoverable only under a hypothesis, ambiguous, substituted, or destroyed;
- where information loss actually occurred.

## Read order before changing code

1. `README.md`
2. `docs/PRIOR_ART.md`
3. `ROADMAP.md`
4. `IMPLEMENTATION_PLAN.md`
5. existing source/tests/research relevant to the task

Do not start by inventing a new plan if the repository already defines the next unfinished milestone.

## Default execution behavior

- Move forward by default; do not stop at analysis or planning when implementation is possible.
- Work on the earliest unfinished dependency-safe task in `IMPLEMENTATION_PLAN.md`.
- Continue through adjacent tasks while tests stay green and no real blocker appears.
- Verification exists to change a decision or prove correctness, not to eliminate every uncertainty.
- Do not report progress as finished merely because documentation, TODOs, or scaffolding were added.
- Prefer a small verified implementation over a large speculative rewrite.
- When facts depend on standards or runtime behavior, verify against authoritative current sources and record the result under `research/`.
- If an assumption is uncertain but non-blocking, choose the conservative implementation, document the assumption, add a test, and continue.

## Required stack and repository shape

Use a simple browser-first TypeScript project unless an existing implementation already establishes an equivalent stack.

Preferred tools:

- TypeScript
- Vite
- Vitest
- npm with committed lockfile
- browser-native APIs where standards-compliant
- mature encoding libraries only where the platform does not provide the required operation
- GitHub Actions for test/build and GitHub Pages deployment

Expected shape:

```text
src/
  core/
  encodings/
  exhibits/
  ui/
fixtures/
research/
tests/
```

Do not create a backend unless a concrete requirement proves static hosting insufficient. The public exhibit should work from GitHub Project Pages.

## Architecture rules

### 1. Bytes and text are different domains

Never represent both as an untagged `string`.

Core types must make illegal transitions difficult. An encode operation accepts text and produces bytes. A decode operation accepts bytes and produces text.

### 2. The trace is the source of truth

Every transformation must create a deterministic trace entry containing enough information to inspect the step independently of the UI.

At minimum preserve:

- operation kind;
- input domain;
- output domain;
- input representation;
- output representation;
- encoding label;
- error mode;
- reversible/loss classification;
- loss reason when applicable;
- warnings/assumptions.

The animation is a view of the trace. The animation must never be the only place where state exists.

### 3. Preserve original bytes whenever possible

Do not claim recovery if original byte identity has already been discarded.

U+FFFD and other substitution paths must be treated as evidence of an information-loss boundary, not as mysterious encoded source characters.

### 4. Reuse standards and mature implementations

Do not maintain a hand-written GBK/Big5/Shift-JIS/Windows-1252 mapping table.

Before implementing legacy encoding behavior, check WHATWG Encoding, Unicode guidance, browser `TextDecoder` behavior, and mature libraries such as ICU/iconv/iconv-lite as appropriate.

Important Web compatibility detail: labels such as ISO-8859-1 may resolve to Windows-1252 semantics under WHATWG. Do not teach a historical encoding identity when the browser API is actually using a compatibility mapping.

### 5. Adjacent layers must stay labeled

Normalization, BOM/endian handling, HTML entities, JSON escaping, URL percent encoding, Base64, filesystem transformations, and font rendering are not all “character encoding”. If included, model them as distinct adjacent transformation layers.

## Loss taxonomy

Use the repository taxonomy consistently:

- `L0 exact`: exact inverse exists with preserved information;
- `L1 recoverable-with-hypothesis`: information remains, but recovery requires choosing the correct historical transformation hypothesis;
- `L2 ambiguous`: multiple histories/originals are consistent with the current state;
- `L3 substituted`: replacement such as U+FFFD or `?` has collapsed source distinctions;
- `L4 destroyed`: deletion, overwrite, lossy normalization/transform, or other irreversible destruction has removed identity.

Do not silently downgrade/upgrade a loss level for prettier UI wording.

## Testing contract

Tests are mandatory for core logic.

Always include or preserve tests for:

- deterministic trace serialization;
- UTF-8 round trip;
- classic UTF-8 → wrong single-byte decode mojibake;
- exact recovery when the wrong decode remained bijective for the relevant bytes;
- at least two distinct invalid byte inputs whose replacement-decoded text is insufficient to recover the original bytes;
- double-mojibake history replay;
- CJK multibyte boundary cases once those adapters exist;
- fatal versus replacement behavior;
- branch comparison from the same bytes under different decoders;
- any loss-level transition rule added or changed.

When practical, compare adapter behavior against the platform/library used in production rather than duplicating its mapping logic in tests.

Before committing a milestone, run the repository's full test suite and production build.

## UI contract

The public UI should emphasize a chain, not a conversion form.

Every exhibit should make it possible to inspect:

- text/code points;
- hex bytes;
- chosen encoding label;
- operation direction;
- error handling;
- current loss level;
- why the step is or is not reversible.

Keyboard navigation and readable text are required. Motion should not be necessary to understand correctness.

Do not hide the underlying state behind a decorative animation.

## Research contract

When implementation depends on current platform behavior or an historical/standards claim, write a short cited note in `research/` containing:

- question;
- authoritative sources;
- observed runtime/platform behavior if tested;
- project decision;
- date checked.

Prefer WHATWG, Unicode, browser/runtime documentation, ICU/iconv documentation, and original project documentation over secondary tutorials.

## Commit discipline

Commit coherent checkpoints, not every keystroke and not one giant unreviewable dump.

Good examples:

- `build: bootstrap browser test harness`
- `feat: add deterministic transformation trace`
- `test: add replacement loss proof vectors`
- `feat: add classic mojibake exhibit`
- `docs: record browser encoding capability survey`

A documentation-only commit is valid when it records research needed for a subsequent implementation decision. Documentation is not a substitute for the implementation milestone itself.

## Stop conditions

Stop or redirect the current implementation if it is becoming:

- a generic converter;
- a one-click mojibake fixer;
- an encoding detector benchmark;
- a hand-maintained character mapping database;
- an animation whose state cannot be reproduced from core logic;
- a recovery tool that discarded original bytes but still claims certainty;
- a page that calls normalization, escaping, fonts, and encoding the same layer.

## Definition of done for the repository

The project is considered substantially complete when all required milestones in `IMPLEMENTATION_PLAN.md` are implemented, tested, documented, deployed to Project Pages, and the major exhibits can be understood without cloning the repository.

Until then, agents should continue implementing the next dependency-safe unfinished task rather than writing a new roadmap.