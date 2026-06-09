---
trigger: glob
globs: **/strings.xml
---

# Android strings.xml correctness

Mechanical correctness for any `strings.xml` you write or edit. A mistake here breaks the
build or breaks runtime formatting silently — so these checks are non-negotiable. The
*judgment* of localization (intent, tone, layout) lives in the `android-string-translator`
skill; this rule only guards the file mechanics.

## File & locale folder
- Translations go in `res/values-<qualifier>/strings.xml`, never in `res/values/`. Create the folder if missing.
- BCP47 qualifiers: `values-de`, `values-ja`, `values-ko`, `values-vi`; region forms `values-zh-rCN` / `values-zh-rTW`, `values-pt-rBR`; script forms `values-b+sr+Latn`.
- Mirror the source file's key order. In partial scope, insert new keys **without** reordering or rewriting existing entries.
- Respect `translatable="false"` — skip those keys entirely; never emit them in a translated file. Brand/product names usually stay in the source language.

## Escaping inside a value
- `'` → `\'` (or wrap the whole value in `"…"`)
- `"` → `\"`
- `&` → `&amp;`   `<` → `&lt;`
- Leading `@` or `?` → `\@` / `\?`
- Newline `\n`; non-breaking space ` `

## Format arguments (most common build/runtime breaker)
- Multiple args **must** be positional: `%1$s … %2$d` — never bare `%s %d`. Bare args break the instant target word order differs from English, which it routinely does in CJK and Arabic.
- Keep every placeholder verbatim and untranslated; only reposition it to read grammatically.
- Mark do-not-translate spans with `<xliff:g id="…">%1$s</xliff:g>` when the source uses it; otherwise keep `%1$s` as-is.

## Markup & plurals
- Wrap real markup (`<b>`, `<u>`, anchor tags) in `<![CDATA[…]]>`.
- Count-dependent text → `<plurals name="…">` with **exactly** the target language's required `quantity` items, not English's two forms:
  - ja / zh / ko / vi → `other` only
  - many Slavic (ru/pl/cs…) → `one / few / many / other`
  - ar → `zero / one / two / few / many / other`
- If the source string isn't plural-safe (a count spliced into a flat string), flag it for a `plurals` resource instead of translating the pieces.

## Never
- No comments inside `strings.xml` — notes, warnings, and flags go to chat or the PR description, not the file (they pollute future diffs and translation passes).
- No silent truncation and no invented abbreviations to make text fit. Surface an overflow as a layout decision instead.
