---
description: Scan the codebase for non-English internal code comments (Turkish etc.) and translate them to English, per the "Internal Code Standards (English)" guideline. Language- and framework-agnostic. Preserves user-facing strings, localization files, and data.
argument-hint: [optional scope — directory path or file glob; omit to scan the whole project]
---

# Translate code comments to English

You are cleaning up non-English **internal code comments** across this project. The governing rule is **Internal Code Standards (English)**: class/type names, methods, variables, constants, database schemas, stored enum/status values, API endpoints, configuration keys, and **internal comments** must all be English. User-facing content (UI labels, validation messages, notification bodies, localization files) is Turkish/multi-locale by design and is **out of scope**.

Scope argument from the user (optional): `$ARGUMENTS`

If `$ARGUMENTS` is empty, scan the whole project. Otherwise treat it as a directory path or glob and restrict the scan to that.

> This command rewrites **comment text only**. It never changes identifiers, strings, or executable code.

## Workflow

Follow these phases in order. Use task tracking (one task per phase) if your agent supports it.

### Phase 1 — Detection (parallel sub-agents)

Dispatch read-only detection sub-agents **in parallel**, split by file-type group so each returns a focused file+line list. Adapt the groups to the stack in use; a typical split:

1. **Backend source** — server-side language files (e.g. `.php`, `.py`, `.rb`, `.go`, `.java`, `.cs`). Comment forms: `//`, `#`, `/* */`, `/** */`, docstrings.
2. **Frontend source** — `.js`, `.ts`, `.jsx`, `.tsx`, `.vue`, `.svelte`, plus root configs. Comment forms: `//`, `/* */`, `/** */`, and JSX `{/* */}`.
3. **Templates / markup** — template engines and HTML (e.g. Blade, Twig, ERB, `.html`). Comment forms: engine comments (`{{-- --}}`, `{# #}`) and `<!-- -->`.

**Exclude everywhere:** dependency dirs (`vendor/`, `node_modules/`), build output, VCS internals, and **auto-generated code** (flag but never edit — regeneration wipes edits).

Each detection agent is **detection only** and returns, grouped by file: path, line number, and the comment snippet.

### Detection heuristics

The team's source language is usually Turkish. Use ripgrep-style patterns on comment lines:

- **Turkish-specific characters in comment lines:**
  `^\s*(//|#|\*|/\*)[^\n]*[çğıöşüÇĞİÖŞÜ]`
- **ASCII-fied Turkish keywords in comment lines (case-insensitive):**
  `\b(icin|ile|olan|ekle|guncelle|kontrol|kayit|sayfa|bilgi|kullanici|yonetimi|goster|gizle|duzenle|ornek|islem|alan|saglar|silinemez|kural|musteri|varsa|zaten|tumu|devam|gerekli|secim|buraya|yazilim)\b`
- **JSX inline comments:** `\{/\*.*[çğıöşüÇĞİÖŞÜ].*\*/\}`
- **Template comments:** match the engine's comment delimiters containing Turkish characters (multiline).

Report path + line + snippet for every hit. If the team's source language is not Turkish, swap the character/keyword set accordingly.

### What does NOT count as a violation (leave alone)

- **String literals** — non-English text inside translation calls, HTML/JSX text children, or any attribute (`alt`, `title`, `aria-label`, `placeholder`). These are user-facing UI.
- **Localization files** — everything under the i18n/translation directories.
- **Locale-label data** — locale names appearing as data or as a docblock `@return` example (e.g. `['tr' => 'Türkçe', ...]`).
- **Seed / fixture data** — non-English category/product/brand names inside data arrays, heredocs, or literals are **data**, not comments.
- **File/path & section references inside an otherwise-English comment** — real identifiers; keep them intact even if they contain non-English words.
- **Auto-generated files.**

### Phase 2 — Review & disambiguate

1. Spot-read 2–3 ambiguous hits yourself to confirm they are comments, not string literals caught by the regex.
2. Watch for **stray/duplicate files** (`* 2.tsx`, `*.bak`, `*.copy.*`). If found, check references before proposing deletion — and **ask** the user first.
3. Show the user a short summary (files grouped by language/extension). If the set is large (> 25 files), ask whether to proceed or narrow scope.

### Phase 3 — Translate comments

For each violating file: read it, then edit each violating comment.
- **Preserve meaning, context, and tone.** Keep bug IDs, file paths, section headings, and technical terms (`super_admin`, package names, framework APIs) intact.
- Use clear, idiomatic English suitable for a senior engineer. Keep roughly the original length — do not expand prose or "clean up" surrounding code.
- Preserve structure: single-line stays single-line, docblock stays docblock.

### Phase 4 — Format & verify

1. **Formatter/linter** — if the project has one, run it on the changed files (real run, not a check-only/dry mode if the project's own convention is to auto-fix).
2. **Tests** — comment-only changes have no runtime surface; a smoke run is enough. Run affected tests only if you touched test files.
3. **Sweep** — re-run the Phase 1 patterns to confirm no comment-line matches remain. Expected residual (NOT violations): string literals, locale-label data, and non-English filenames/section titles inside English comments.
4. **Git status** — should show exactly the files you edited plus any approved deletions.

### Phase 5 — Report

Summarize: files changed (by language), files deleted (and why), verification output, and an explicit list of residual non-English matches you deliberately kept, one-line justification each.

Then **ask** the user how to finalize. Do not auto-commit.

## Finalizing (only when the user approves)

- Commit message in **Turkish**; no AI watermarks, tool names, or attribution footers — only the clean technical description.
- For a large change, prefer a dedicated branch + pull request so the user can review the diff, rather than committing to the default branch.

## Important rules to respect

- **Never edit localization files** — those are user-facing translations managed elsewhere.
- **Never touch UI text or markup attributes** — moving them to i18n is a separate task.
- **Never change identifiers** (class/method/variable/file names). This command rewrites comment text only.
- **Skip and never edit auto-generated files** — regeneration will wipe edits anyway.
- **Preserve filename and section-title references** even when they contain non-English words.
- **Ask before deleting** any stray/duplicate file.
