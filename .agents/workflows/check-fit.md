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

## Step B — Prefer a view-layer fix (the default)

When a HARD/SOFT site won't fit, recommend a **view fix**. Don't default to "scale it down" —
walk the families in order (space → reflow → scale → trim-display → redesign) and pick per the
string's **class** (the table below). Always name the concrete API for the project's UI system.

**B1. Give the text more room** (best: nothing about the text changes)
- Wrap / +1 line: Compose `maxLines = 2` · View `android:maxLines="2"` (drop `singleLine`)
- Widen/flex the container: `weight()` / `widthIn` · `0dp`+weight, `app:layout_constrainedWidth="true"`, larger `maxWidth`
- Let chips/tabs flow or scroll: Compose `FlowRow`, `ScrollableTabRow` · View `app:tabMode="scrollable"`, `HorizontalScrollView` row
- Scrollable container for long body: Compose `Modifier.verticalScroll(...)` · View `ScrollView`/`NestedScrollView` — the standard fix for dialog/legal/onboarding body
- Per-locale spacing: smaller padding/margins via `values-<lang>/dimens.xml`

**B2. Reflow the same glyphs** (typography-level, helps long-word languages)
- Hyphenation: Compose `TextStyle(hyphens = Hyphens.Auto)` · View `android:hyphenationFrequency="normal"` — big win for DE/FI compounds
- Balanced line breaking: Compose `LineBreak.Paragraph` · View `android:breakStrategy="balanced"`
- ` ` to keep number+unit together; soft hyphen `­` at sane break points; slightly tighter `letterSpacing`

**B3. Scale the text — WITH A FLOOR** (the only family the lazy answer uses; never the only offer)
- Compose `TextAutoSize.StepBased(minFontSize = …)` (1.8+) · View `autoSizeTextType="uniform"` + **`autoSizeMinTextSize`**
- **Legibility floor: ~12sp body / ~10sp caption-chip.** If text only fits *below* the floor, autosize is the **wrong tool** — go back to B1/B2 or escalate. Never let payment/price/legal text shrink below the floor.

**B4. Trim what's displayed** (string stays intact in resources; only rendering is cut — class-gated, see table)
- Ellipsis end/start/middle: Compose `TextOverflow.Ellipsis` / `StartEllipsis` / `MiddleEllipsis` (1.8+) · View `android:ellipsize="end|start|middle"` — for **dynamic user content only** (file names, URLs, notes); middle keeps both ends of paths/emails visible
- Marquee for ambient one-liners: Compose `Modifier.basicMarquee()` · View `ellipsize="marquee"` — track titles, tickers; never on interactive/critical text
- "Read more" expansion for long marketing/body blocks

**B5. Redesign the element** (a design change — flag it)
- Icon-only or icon + tooltip for chips/tabs; move text to a subtitle; restructure the row

**Class → allowed techniques** (recommend per the string's Stage-2 class):

| String class | Prefer | Allowed | **Never** |
|---|---|---|---|
| CTA / button | B1 wrap-2 / widen → B3 floor | B5 | **ellipsis, marquee, below-floor scale** |
| Tab / chip / badge (HARD) | B3 floor → B1 scrollable tabs/flow | B5 icon+label | wrap (breaks the bar), ellipsis |
| Title / header | B1 wrap +1 → B2 hyphenation | B3 mild | ellipsis hiding meaning |
| Body / dialog / onboarding | B1 wrap + scroll container | B2 | scale below floor |
| Legal / payment / price | B1 wrap + scroll | — | **any trim, marquee, below-floor scale** |
| Dynamic content (names, URLs, notes) | B4 middle/end ellipsis | B1 | — |
| Notification / ticker | B4 marquee / system 2-line | B1 | — |

If nothing in B1–B4 fits the class → Step C (shorten, dev-approved) or Step B5/escalation.

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
- the constraint read (or assumed) + tier + the string's class;
- verdict: fits / at risk / overflows;
- **at least two viable fixes from different B-families, ranked for that class, each with the
  concrete API** (e.g. for a button: `maxLines=2` (B1) → `TextAutoSize.StepBased(minFontSize=12.sp)`
  (B3) — never just "autosize"). One-option reports are an anti-pattern;
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
  B  view-fix families, walk in order + gate by class:
       B1 room (wrap/widen/flow/scrollable-tabs/scroll-container/locale-dimens)
       B2 reflow (hyphenation, balanced breaks, nbsp/soft-hyphen)
       B3 scale WITH floor (~12sp body/10sp chip; below floor = wrong tool; never price/legal)
       B4 trim display (ellipsis end/start/middle, marquee) — dynamic content only, never CTA/legal
       B5 redesign (icon+tooltip, restructure) — flag as design change
  C  last resort (dev-approved): shorten form, intent intact; register guard; never truncate/invent
  D  report per key: class + ≥2 ranked fixes from different families w/ concrete APIs — never scale-only
```
