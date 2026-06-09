---
name: android-string-translator
description: Use this skill when a coding agent runs inside an Android Kotlin/Compose app repository and is asked to localize or translate user-facing strings into a non-English language. Triggers include "localize this app", "translate the strings", "dịch string", "add German/Japanese/Korean", "translate just these new keys", "localize the strings I added", "fix awkward translations", or any request to populate values-xx/strings.xml — whether for the whole file or only a few new/changed keys. The agent reads the Compose UI and code directly to derive real layout constraints, responsive/line behavior, font coverage, and on-screen context before translating — so output preserves the designer's intent, matches tone, and does not break layout. Do NOT use for long-form prose or non-Android projects.
---

# Android String Localizer

A model-agnostic framework for a coding agent to localize an **Android Kotlin/Compose** app
from inside its repository. Translate by **intent and tone**, not word-by-word, while
respecting the real layout limits you read from the code.

This skill is the **expertise**. Two companions carry the rest:
- **Rule `android-xml`** auto-applies whenever a `strings.xml` is in context and owns the mechanical correctness (escaping, positional args, plurals syntax, locale folders). Don't restate it here — trust it.
- **Workflow `/localize-app`** runs the stages below in order, with confirm-gates and the git/grep commands. Use it for a guided run; the stages below are the knowledge it draws on.

## Core principle — fidelity first, naturalness as the lever

> Preserve the **designer's intent** exactly. Adapt only *how* it's said, never *what* is said.
> When a literal translation reads stiff or crude, adjust **register and energy** to match the original — not the meaning, scope, or content.

A button labeled "Done" is a promise of completion, not the adjective "done" — express that
*same intent* naturally. You are not rewriting, improving, or re-scoping copy.

**Hard line — never, even if "better":** don't add info/claims the source lacks; don't change
a CTA's action or outcome; don't soften/strengthen/re-scope; don't inject personality the
source doesn't carry; don't merge/split/reorder beyond what target grammar strictly requires.
The adjustable lever is **tone/register/word energy**. The fixed payload is **meaning + intent + scope**.

---

## STAGE 0 — Detect current voice, then confirm

Infer tone from the app's existing strings; don't ask cold.

Read across `res/values/strings.xml` + Compose `Text` usages and note: sentence length,
punctuation/emoji habits, person/address, warmth, vocabulary level, error/CTA phrasing. Draft:

```
DETECTED VOICE CARD (inferred)
- Adjectives: [3-4]
- Formality: [e.g. neutral; addresses user as "bạn"/du/です-form]
- On-brand:  "<a real source string>"
- Off-brand: "<an over-expressive counterexample>"
- Hard rules observed: [emoji? exclamations in errors? slang?]
- Confidence: [high / mixed — note inconsistency]
```

If source strings are inconsistent in tone, say so — it's a finding. Then ask once: **keep
this tone or shift it** (warmer/more formal/playful)? KEEP → lock as-is; SHIFT → update card
(the one sanctioned divergence from source tone). A dev-supplied voice card is used verbatim.
The locked card is the **ceiling on personality** for every string and language.

---

## STAGE 1 — Read the code to get context & constraints (codebase mode)

Never treat strings as a flat list. Trace each string to where it renders and read the Compose code.

**Read economy — read narrow by default; cost scales with what you open.**
- Grep for the **exact keys** in scope (`R.string.<key>`), never broad/fuzzy scans of the repo.
- Open **only the render-site file(s)** grep returns, and within them read only the composable around the match (a range, not the whole file) — most layout signals sit within ~20 lines of the `Text`.
- Read `values/strings.xml` once; for partial scope read only the target keys + any glossary terms they reuse, not the whole resource set.
- Skip files that can't change the budget (tests, build files, unrelated screens). Don't rasterize screenshots unless a constraint is genuinely unresolvable from code.
- A dev-supplied **key list + file path** is the cheapest path — when given, trust it and skip discovery. If absent, the grep-then-narrow approach above is the fallback; ask for the path only when a key renders in many sites and the tightest one is unclear.

**1. Trace.** `grep -rn "R.string.key\|stringResource(R.string.key"`. A key may render in
several composables with different limits — localize for the **tightest**.

**2. Extract the budget & shape from Compose:**
- `Modifier.width/widthIn/sizeIn`, `fillMaxWidth`, `weight()` (shared space with siblings)
- `maxLines`, `overflow = TextOverflow.Ellipsis`, `softWrap`
- autosize (`BasicText` with `autoSize` / `TextAutoSize`) — if present, mild overflow is absorbed
- container: `Row`/`Column`/`NavigationBarItem`/`AssistChip`/`Button`/`TopAppBar`/`AlertDialog` — the widget confirms function (Stage 2) and resolves ambiguity the key name can't

Decide tier from **actual code**: **HARD** (single line, fixed/narrow width, no autosize —
tabs, chips, badges, segmented buttons), **SOFT** (one line but shrinks/scrolls — titles,
toasts), **FLEXIBLE** (wraps — dialog body, onboarding).

**3. Responsive & line behavior:** locked to one line or wraps? autosize present? in a
`weight`/flex row sharing space? any `values-swXXXdp` / orientation / config-qualified
resources with different budgets (localize for the most constrained)? **Strings built by
concatenating fragments in Kotlin** (`a + count + b`) → FLAG: assembly breaks word
order/gender/plurals; recommend a single parameterized `plurals`/format resource — don't
translate the pieces.

**4. Feed findings into Overflow Step A (budget), Stage 2 (class), Stage 0 (voice).**

**Agent boundaries:** read freely; writing translations back is side-effectful — write only to
`values-<lang>/strings.xml`, show the diff, never edit source-language strings, keys, or Kotlin
logic. Surface fragment-assembly / missing-plural / bad-source problems as code recommendations
for the dev, not patched over in translation.

---

## STAGE 2 — Classify each string

| Class | Rule |
|---|---|
| CTA / button | Imperative, shortest natural verb; match action energy. |
| Title / header | Noun phrase, native capitalization (most languages ≠ English Title Case). |
| Error / warning | Clear, blame-free, actionable; never literal jargon (translate intent of errno/codes). |
| Empty state | Light, encouraging within the voice card; don't add hype. |
| Onboarding / marketing | Persuasive, idiomatic; never add claims or change the offer. |
| Label / hint / placeholder | Functional, conventional per language. |
| System / confirmation | Match severity; destructive actions unambiguous. |
| Notification / toast | Concise, often verbless in target. |
| Legal / payment / security | Precise, formal; accuracy > naturalness; no loose paraphrase. |

---

## STAGE 3 — Translate by intent + validate

**3a. Target-language profile (infer once per language, reuse):** formality default for app
UI; expansion ratio (DE/FI/RU/PL +30-35%; FR/ES/PT/IT +15-25%; JA/ZH/KO usually shorter; AR
variable + RTL); plural form count (EN 2; many Slavic 3-4; JA/ZH/VI/KO 1; AR 6 → use
`plurals`); gender/agreement; script/direction; native capitalization & punctuation.

**3b. Translate for the function** applying the voice card — tighter/literal for
legal/error/system, more idiomatic for marketing/empty-state — intent, scope, content fixed
throughout.

**3c. Hard checks (judgment layer; the `android-xml` rule enforces the file mechanics):**
1. **Variables** — every `%s %d %1$d {name}` preserved, untranslated, repositioned to be grammatical.
2. **Plurals** — count-dependent → emit a `plurals` resource with the target's required forms; if the source isn't plural-safe, flag it rather than fake it.
3. **Length / layout** — estimate vs. budget × expansion ratio; if a HARD/SOFT element risks overflow, run the Overflow protocol. Never silently truncate or invent abbreviations.
4. **Glossary** — same source term → same target term everywhere; lock product/feature/brand names. **Polysemy guard:** if one source word appears with different functions (e.g. tab "Clean" = remove junk vs. "all clean" = state), detect it and decide deliberately whether to unify or split the term; record both.
5. **Tone/register** — reads native at the voice card's energy level, not dictionary-literal.
6. **Intent fidelity** — same meaning, scope, action; nothing added/softened/strengthened/re-scoped.

**3d. Self-critique:** each string must pass BOTH *"carries the designer's exact intent?"* AND
*"native & on-voice in-app?"*. If they conflict, keep fidelity and flag the tension.

---

## Overflow & layout protocol

Shorten the **form**, never the intent; when impossible, escalate as a design decision.

### Step A — Budget
1. **From Compose code** (source of truth): the constraints in Stage 1 for the string's tightest render site.
2. **From config** if present (`maxLength`, dimens, qualified resources).
3. **From widget type** if unknown: HARD / SOFT / FLEXIBLE tiers above.
4. Still unknown for a HARD element → assume source length ≈ budget, flag it, ask the dev for a char/width limit if it matters.
Apply the language expansion ratio to predict risk before committing.

### Step A2 — Font fallback risk (CJK/Thai/etc.)
Android renders with the app's `FontFamily` only if that font has glyphs for the target
script. A custom font that covers Latin but **not** KR/JA/ZH/Thai/Devanagari triggers
**per-glyph system fallback** (Noto etc.) at runtime — different metrics, usually wider/taller
line height → measured width and vertical fit shift, often beyond the expansion-ratio estimate.

When the target language uses a script the app font may not cover:
1. Find the font setup — `Type.kt`, `FontFamily(Font(R.font.…))`, `Typography`, any `fontFamily =` on `Text`. Note whether a CJK/target-script face is declared.
2. If coverage is unconfirmed: **treat HARD-element budgets as softer than the glyph count suggests** (add margin) and **flag vertical metrics too** (fallback line height can clip in fixed-height rows), not just width.
3. Recommend to the dev: bundle a target-script-capable font (e.g. Noto Sans CJK/KR/JP) or set an explicit `FontFamily` fallback chain, and verify the tight screens on-device for that locale.
4. Do not "solve" fallback by over-shortening the translation — that trades a real intent for a guessed metric. Escalate per Step C instead.

### Step B — Shorten the form (stop at first that fits with intent intact)
1. Shorter exact synonym at the same register.
2. Drop redundancy the UI already supplies (button in a "Delete files?" dialog → just "Delete"). Preferred, not a compromise.
3. Native grammatical short form (idiomatic compaction natives actually use).
4. Established, locale-correct abbreviation **only if genuinely conventional** — never invented.
5. Icon + minimal label (a design change — flag it).

**Register guard:** for a HARD element, if the only fitting candidate shifts register or picks
an off-meaning word (eval lesson: tab "Clean" → DE "Säubern" drifts toward household-cleaning),
do **not** accept the weak word — escalate to Step C. A correct-but-overflowing word + a design
fix beats a fitting-but-wrong word.

### Step C — When it won't fit without losing meaning (design problem)
Surface to the dev, ranked: allow wrap / +1 line for this locale; autosize (`TextAutoSize` /
note legibility floor); widen/flex the container for long-language locales; icon-only or
icon+tooltip; documented locale-specific compromise for approval.
Never: mid-word truncation, ellipsis hiding required info, invented abbreviations, meaning changed to fit.

### Step D — Output per flagged string
Best natural translation; fitted version (if different) + which budget; the constraint
known/assumed + which Step-B technique used; font-fallback note if applicable; escalation note
+ recommended option if it hit Step C.

---

## Glossary protocol
Build a term table (source → target → rationale); lock product/feature/brand names and
recurring UI nouns; reuse on every occurrence (no synonym drift); apply the polysemy guard;
output the table for reuse. A dev-provided glossary is authoritative.

## Anti-patterns
Word-by-word ignoring function; copying English Title Case; translating inside placeholders;
loose paraphrase of legal/security; "improving" the source; changing action/severity/scope;
sounding more expressive than the voice card; silent truncation; over-shortening to dodge a
font-metric guess; concatenating fragments instead of a parameterized resource; coining a new
term for something already translated elsewhere. (File-level anti-patterns — bare `%s`,
unescaped `'`/`&`/`<`, comments in XML, auto-writing without a diff — are enforced by the
`android-xml` rule.)
