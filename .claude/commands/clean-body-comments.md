---
description: Scan the codebase for multi-line explanatory comment blocks placed inside function/method bodies and remove them, per the "Comments Inside Functions and Methods" guideline. Deletes redundant narration, relocates genuine "why" into the standard doc block, collapses in-place rationale to one short line. Language- and framework-agnostic. Comment-only — never changes executable code.
argument-hint: [optional scope — directory path or file glob; omit to scan the whole project]
---

# Clean up in-body comment narration

You are enforcing the **"Comments Inside Functions and Methods"** guideline: do not place explanatory comment blocks between executable statements inside function/method/closure bodies. The home for the "why" is the language's standard documentation block (docstring / JSDoc / PHPDoc / XML-doc), the architectural docs, an ADR, the changelog, or the commit message — never narration wedged between statements.

Scope argument from the user (optional): `$ARGUMENTS`

If `$ARGUMENTS` is empty, scan the whole project. Otherwise treat it as a directory path or glob and restrict the scan to that.

> This command is **comment-only**: it never changes executable code, control flow, identifiers, strings, or the formatting of executable lines. It only removes, relocates, or collapses comments.

## What counts as a violation

A run of **2+ consecutive comment lines** (`//`, `#`, or a `/* */` / `{/* */}` block) that sits **inside a function/method/closure body** (after the opening brace, between or before statements) and narrates rationale, decision-making, implementation history, edge cases, race conditions, or step-by-step reasoning.

## What is NOT a violation — LEAVE UNTOUCHED

- **Documentation blocks ABOVE a declaration** (docstring / JSDoc / PHPDoc / XML-doc on a class/method/function/property/type). This is the approved home for context — never touch these.
- **A single short one-line inline comment.** The rule explicitly allows one short inline comment. Do not remove single-line comments unless clearly part of a removed block.
- **Commented-out CODE** (disabled statements) — a separate concern.
- **User-facing strings**, localization keys, template/JSX text, markup attributes.
- **Comments directly above a declaration** that document the thing below — approved "context above the thing" placement, even when written as line comments.
- Annotation/directive comments (`@param`, `@return`, type hints, `eslint-disable`, linter/analyzer pragmas).
- Single-line section labels / data labels / ASCII diagrams in tests & fixtures.

## Workflow

Follow these phases in order; track progress with one task per phase (and, in Phase 3, one task per edit-area).

### Phase 1 — Detection (parallel sub-agents)

Dispatch read-only detection sub-agents **in parallel**, split by area (e.g. backend application code / backend tests+fixtures+config / frontend source). Narrow to the areas the `$ARGUMENTS` scope actually touches.

**The signal is runs of 2+ consecutive comment lines, then a placement judgement.** A pure regex cannot tell "inside a body" from "above a declaration" or "commented-out code" — the agent must READ around each hit. A consecutive-comment reducer:

```
rg -n '^\s*(//|#)' <scope> | awk -F: '{k=$1; l=$2; if(k==pk && l==pl+1){if(!(k in s)){print pk":"pl; s[k]=1} print k":"l} pk=k; pl=l}'
```

Also find `/* */` / `{/* */}` blocks not attached to a declaration. For each candidate, classify: **in-body narration** (violation), **above-declaration doc** (leave), or **commented-out code** (leave). Return, grouped by file, each violation's line range + first-line snippet + one-line reason; put uncertain cases in a **BORDERLINE** section with reasoning. Detection only — no edits.

### Phase 2 — Review, scope decision & branch

1. Spot-read 2–3 borderline hits yourself to confirm the classification.
2. Show the user a short summary (blocks and files, grouped by area).
3. **This rule is often violated pervasively** — hundreds of blocks is normal. Do NOT silently mass-edit a large set; confirm scope/approach with the user (whole codebase vs. production code only vs. a pilot file first).
4. **Prefer a dedicated branch for any non-trivial run** so the user reviews via diff/PR rather than on the default branch:
   ```
   git fetch && git status
   git checkout -b refactor/in-body-comments
   ```
   Do this before editing.

### Phase 3 — Fix (parallel edit sub-agents, one shared spec)

Write the **fix specification once** to a scratch file, then dispatch edit sub-agents **in parallel over DISJOINT file sets** (disjoint = no edit conflicts). Each agent reads the same spec plus its own flagged-locations list.

**Fix philosophy — decide per block, in priority order:**

1. **DELETE if redundant.** If the code below is already self-explanatory, delete the narration. Most common correct fix.
2. **RELOCATE genuine "why" to the doc block.** If the block explains a non-obvious WHY (a real tradeoff, an incident guard, a race-condition reason, a security rationale) AND the enclosing unit is a **named method/function** with (or able to have) a doc block above it, move a **concise** version (1–2 sentences) there. Preserve real identifiers (issue numbers, ADR refs); drop pure change-history framing.
3. **COLLAPSE to one short inline line** when the clarification is genuinely needed *at that exact spot* and cannot be relocated (test closures, anonymous callbacks, setup/migration blocks — no named-declaration home). Reduce the block to a single concise line, or delete if even one line adds nothing.
4. **Change-history narration → DELETE.** "migrated from X", "Production incident: …", "used to be … now …" is exactly what the rule forbids. Keep at most a one-line behavioral fact if still relevant.

**Placement-specific guidance:**
- **Imperative method bodies** (controllers, services, handlers, observers, jobs): prefer rule 2 (relocate to doc block), else rule 1 (delete).
- **Declarative / builder-style config** (form/table/schema builders, fluent chains) with no per-element doc home: rule 1 (delete) or rule 3 (collapse). Do not convert comments into config API calls — that is a separate change.
- **Tests, fixtures, migrations, bootstrap/setup closures:** rule 3 (collapse) or rule 1 (delete). Do NOT invent doc blocks above test-case functions. Keep ONE short line for a real footgun (e.g. a race-condition workaround); delete step-by-step narration and post-mortem prose.
- **Frontend component/hook bodies:** prefer rule 2 (relocate to a doc block above the component/hook/function) or rule 1 (delete); rationale tied to a specific effect/memo → collapse to one line only if genuinely needed.

**Hard constraints for every edit agent:**
- Comment-only. Executable code byte-identical afterwards, except removed/edited comment lines and any doc block added above a declaration.
- Never touch commented-out code, auto-generated files, or localization files.
- Preserve linter/analyzer directives and real identifiers exactly.
- If a block is genuinely borderline (unsure body-vs-declaration), LEAVE IT and note it.
- Report per-file counts (deleted / relocated / collapsed) and anything deliberately left.

### Phase 4 — Format & verify

1. **Formatter/linter** — run the project's formatter on the changed files if it has one.
2. **Comment-only proof — the key safety check.** Every added/removed diff line must be a comment or blank; this must print **nothing**:
   ```
   git diff --no-color | grep -E '^[+-]' | grep -vE '^(\+\+\+|---)' \
     | sed -E 's/^[+-][[:space:]]*//' \
     | grep -vE '^(//|#|\*|/\*|\*/|\{/\*|\*/\}|$)'
   ```
   Any output is an executable line that was touched — investigate before proceeding.
3. **Tests.** Comment-only changes have no runtime surface: if the formatter passed and the comment-only proof is empty, a full suite run is not required. Say so rather than skipping silently. If unsure the change is truly comment-only, run the affected tests.
4. **Re-sweep.** Re-run the Phase 1 reducer over touched areas. Remaining 2+ comment clusters are EXPECTED and NOT violations when they are: commented-out code, above-declaration notes, a one-line rationale followed by a linter directive, or type-member docs. Confirm each residual is one of these.
5. **Git status** — should show exactly the files you edited.

### Phase 5 — Report

Summarize: files changed and blocks handled (grouped by area), breakdown by action (deleted / relocated / collapsed), verification (formatter result, the empty comment-only proof, re-sweep residual explanation), and any borderline blocks deliberately left with a one-line reason.

Then **ask** the user how to finalize. Do not auto-commit.

## Finalizing (only when the user approves)

- Commit message in **Turkish**; no AI watermarks, tool names, or attribution footers — only the clean technical description.
- **Large change (default here):** commit on the dedicated branch, push, and open a pull request so the user reviews in the browser. Do not merge unless asked.
- In-body comment cleanup is **comment-only with no user-visible effect** → it does not warrant a changelog entry. Say so rather than silently omitting.

## Important rules to respect

- **Comment-only, always.** Never change executable code, identifiers, markup, hook deps, config values, assertions, or test steps.
- **Never invent doc blocks above test-case functions** — collapse or delete instead.
- **Never touch commented-out code, auto-generated files, or localization files.**
- **Preserve genuine context, do not just delete it** — relocate real "why" into the doc block or a single concise line. Losing a load-bearing fact is worse than leaving a block.
- **Keep real references** — issue numbers, ADR refs, doc-path pointers. Drop only the historical framing around them.
- **Above-declaration line-comment notes are out of scope** — they are correctly placed. Do not mass-convert them.
