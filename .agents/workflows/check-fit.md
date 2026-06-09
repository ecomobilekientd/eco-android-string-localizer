---
description: Audit whether on-screen text fits its UI and recommend view-layer fixes for the strings that don't. Works for Compose and the View/XML layout system. Run after translation (or whenever a designer changes copy) on the keys/screens flagged as overflow-risky. Use when asked to "check overflow", "does this fit", "kiểm tra tràn chữ", "check layout fit", or after /localize-app flags tight strings.
---

# /check-fit

Resolve the **fit** concern that the `android-string-translator` skill deliberately does **not**
own. Core stance: **fitting text is the view layer's job** — an auto-sizing component or a wider
container — not something solved by mangling the translation. So this workflow's default output
is a *view-layer recommendation*, and shortening the wording is a last resort with dev approval.

Input: a set of keys/screens to check (from `/localize-app`'s flags, a git diff, or a dev list)
plus the target language(s). If none given, ask which screens/locales to audit.

---

## Step A — Establish the real budget (read the constraint, don't guess)

For each flagged key, open its tightest render site and read the actual limit.

**Compose:**
- width: `Modifier.width/widthIn/sizeIn`, `fillMaxWidth`, `weight()` (space shared with siblings)
- lines: `maxLines`, `softWrap`, `overflow = TextOverflow.Ellipsis`
- autosize: `BasicText` + `TextAutoSize`/`autoSize` — if present, mild overflow self-absorbs
- container: `Tab` / `NavigationBarItem` / `AssistChip` / `Button` / `TopAppBar` / `AlertDialog`

**View/XML:**
- width: `android:layout_width` (fixed `dp` vs `wrap_content` vs `0dp`+weight), `minWidth`/`maxWidth`
- lines: `android:maxLines` / `android:lines` / `android:singleLine`, `android:ellipsize`
- autosize: `android:autoSizeTextType="uniform"` (+ `autoSizeMinTextSize`/`MaxTextSize`/`StepGranularity`), or `TextViewCompat.setAutoSizeTextTypeWithDefaults`
- container: `Toolbar` / `BottomNavigationView` / `TabLayout` / `Chip` / `<Button>`
- also check `dimens.xml` and config-qualified resources (`values-swXXXdp`, `-land`) for different budgets — audit the most constrained.

Tier it: **HARD** (single line, fixed/narrow, no autosize), **SOFT** (one line, shrinks/scrolls),
**FLEXIBLE** (wraps). HARD is where overflow actually bites. Apply the language expansion ratio
(DE/FI/RU/PL +30-35%; FR/ES/PT/IT +15-25%; JA/ZH/KO often shorter; AR variable + RTL) to predict
risk before deciding.

## Step A2 — Font fallback risk (CJK / Thai / Arabic / Devanagari …)

Android renders with the app's `FontFamily` only if that font has glyphs for the target script.
A font that covers Latin but **not** the target script triggers **per-glyph system fallback**
(Noto etc.) at runtime → different metrics, usually wider/taller line height, and **loss of the
custom typeface's look** (e.g. a serif display title falls back to a sans Noto). The shift often
exceeds the expansion-ratio estimate.

- Find the font setup — Compose `Type.kt` / `FontFamily(Font(R.font.…))` / `Typography` / Google-Fonts `GoogleFont.Provider`; View `fontFamily` in styles or `app:fontFamily`. Note whether a target-script face is declared.
- If coverage is unconfirmed: treat HARD budgets as **softer than the glyph count suggests** (add margin) and **flag vertical metrics too** (fallback line height can clip in fixed-height rows), not just width.
- Recommend: bundle a target-script font (Noto Sans CJK/KR/JP/Thai/Arabic) or set an explicit fallback chain; verify the tight screens on-device for that locale.
- Never "solve" fallback by over-shortening the translation — that trades a real intent for a guessed metric.

## Step B — Prefer a view-layer fix (the default, ranked)

When a HARD/SOFT site won't fit, recommend a **view fix** first — this is the right owner:
1. **Auto-sizing text** — Compose `TextAutoSize` / View `autoSizeTextType="uniform"` (note a legibility floor for the min size).
2. **Allow wrap / +1 line** for this locale (or `values-<lang>` dimen override).
3. **Widen / flex the container** for long-language locales (`weight`, `wrap_content`, larger `maxWidth`).
4. **Icon-only or icon + tooltip** for chips/tabs where text can't fit at all.

## Step C — Last resort: shorten the *form* (only with dev approval, intent intact)

Only if the dev declines a view fix, and stopping at the first that fits **without** changing
meaning, scope, or register:
1. Shorter exact synonym at the same register.
2. Drop redundancy the UI already supplies (a button inside a "Delete files?" dialog → just "Delete").
3. Native grammatical short form natives actually use.
4. Established, locale-correct abbreviation **only if genuinely conventional** — never invented.

**Register guard:** for a HARD element, if the only fitting candidate shifts register or picks an
off-meaning word (eval lesson: tab "Clean" → DE "Säubern" drifts toward household-cleaning), do
**not** accept it — keep the correct-but-overflowing word and push the view fix instead.
**Never:** mid-word truncation, ellipsis hiding required info, invented abbreviations, meaning changed to fit.

## Step D — Report per flagged string

`key (locale) @ render-site` →
- the constraint read (or assumed) + tier;
- verdict: fits / at risk / overflows;
- recommended **view fix** (Step B) as the primary action;
- font-fallback note if applicable;
- a shortened candidate **only** if the dev opted into Step C, labelled as such.

Keep faithful translations untouched in `strings.xml`; fit recommendations and any approved
shortened forms are proposed to the dev, then applied via `/localize-app`'s write gate.

---

### Quick reference
```
/check-fit → keys/screens + locale(s)?
  A  read real budget at tightest site (Compose: width/maxLines/autosize/weight;
       View/XML: layout_width/maxLines/ellipsize/autoSizeTextType) → tier HARD/SOFT/FLEXIBLE
  A2 target script vs app font → fallback? flag width AND vertical metrics; recommend Noto/bundle
  B  PREFER view fix: autosize → wrap/+1 line → widen container → icon+tooltip
  C  last resort (dev-approved): shorten form, intent intact; register guard; never truncate/invent
  D  report per key: constraint, verdict, view-fix first, font note, optional shortened candidate
```
