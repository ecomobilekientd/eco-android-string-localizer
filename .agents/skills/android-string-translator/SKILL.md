---
name: android-string-translator
description: Use this skill when a coding agent runs inside an Android app repository (Kotlin/Compose OR the View/XML layout system) and is asked to localize or translate user-facing strings into a non-English language. Triggers include "localize this app", "translate the strings", "dịch string", "add German/Japanese/Korean", "translate just these new keys", "localize the strings I added", "fix awkward translations", or any request to populate values-xx/strings.xml — whether for the whole file or only a few new/changed keys. The agent reads the UI code (Compose `Text` or `TextView`/XML) to classify each string's function and on-screen context, then translates to preserve the designer's intent and tone. It flags overflow risks but defers layout-fitting to the `/check-fit` workflow (fitting is the view layer's job, not the translation step's). Do NOT use for long-form prose or non-Android projects.
---

# Android String Localizer

A model-agnostic framework for a coding agent to localize an **Android** app from inside its
repository — whether the UI is **Compose** or the **View/XML layout** system. Translate by
**intent and tone**, not word-by-word, staying aware of where each string renders.

**Division of labor — translation owns the *words*, not the *fit*.** This skill translates
faithfully and *flags* anything that may overflow. It does **not** shorten text or make layout
decisions to force a fit — that is the view layer's job (an auto-sizing component, a wider
container), handled by the `/check-fit` workflow. (This is the architectural line a real dev
will hold you to: don't bake fitting logic into the translation step.)

This skill is the **expertise**. Three companions carry the rest:
- **Rule `android-xml`** auto-applies whenever a `strings.xml` is in context and owns the mechanical correctness (escaping, positional args, plurals syntax, locale folders, markup/CDATA, comments). Trust it; don't restate it here.
- **Workflow `/localize-app`** runs the stages below in order, with confirm-gates and the git/grep commands. Use it for a guided run.
- **Workflow `/check-fit`** owns the overflow & layout-fit protocol (budget, autosize, font fallback, escalation) for both Compose and View/XML. Translation flags; `/check-fit` resolves.

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

Read across `res/values/strings.xml` + the render sites (Compose `Text` or `TextView`/XML) and note: sentence length,
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

## STAGE 1 — Read the code for context (codebase mode)

Never treat strings as a flat list. Trace each string to where it renders and read the UI code
— enough to know **what it is** (function → Stage 2) and **whether it sits somewhere tight**
(→ flag for `/check-fit`). You are gathering *context*, not solving layout here.

**Read economy — read narrow by default; cost scales with what you open.**
- Grep for the **exact keys** in scope (`R.string.<key>` / `@string/<key>`), never broad/fuzzy scans of the repo.
- Open **only the render-site file(s)** grep returns, and within them read only the ~20 lines around the match — most layout signals sit right by the `Text` / `TextView`.
- Read `values/strings.xml` once; for partial scope read only the target keys + any glossary terms they reuse.
- Skip files that can't change the picture (tests, build files, unrelated screens). Don't rasterize screenshots unless a constraint is genuinely unresolvable from code.
- A dev-supplied **key list + file path** is the cheapest path — trust it and skip discovery.

**1. Trace.** A key may render in several places with different limits — note them all; the
tightest one matters for `/check-fit`.
- Compose: `grep -rn "R.string.key\|stringResource(R.string.key"`
- View/XML: `grep -rn "@string/key"` across `res/layout*/` and `getString(R.string.key)` in Kotlin/Java.

**2. Read the render site to confirm function & spot tight spots:**
- **Compose** — the container/widget tells the function: `Button`/`NavigationBarItem`/`AssistChip`/`TopAppBar`/`AlertDialog`/`Tab`. Glance at `maxLines`, fixed `width`/`weight`, `TextOverflow.Ellipsis`, autosize (`BasicText` + `TextAutoSize`).
- **View/XML** — the element/parent tells the function: `<Button>`, `<TextView>` in a `Toolbar`/`BottomNavigationView`/`Chip`/`TabLayout`. Glance at `android:maxLines`/`android:lines`, `android:layout_width` (fixed vs `wrap_content`/`0dp`), `android:ellipsize`, `android:autoSizeTextType`.

You only need a **rough tier** to decide whether to flag: **HARD** (single line, fixed/narrow,
no autosize — tabs, chips, badges), **SOFT** (one line, shrinks/scrolls — titles, toasts),
**FLEXIBLE** (wraps — dialog body, onboarding). Don't compute exact budgets — that's `/check-fit`.

**3. Flag, don't fix.**
- A HARD/SOFT render site + a long-expanding target language (DE/RU, or a script needing font fallback like CJK/Thai/AR) → **flag the key for `/check-fit`**. Keep translating faithfully meanwhile.
- **Strings built by concatenating fragments** (`"You have " + n + " items"`) → FLAG: assembly breaks word order/gender/plurals; recommend one parameterized `plurals`/format resource — don't translate the pieces.

**Agent boundaries:** read freely; writing translations back is side-effectful — write only to
`values-<lang>/strings.xml`, show the diff, never edit source-language strings, keys, layouts,
or Kotlin/Java logic. Surface fragment-assembly / missing-plural / bad-source / overflow-risk
problems as recommendations for the dev (or for `/check-fit`), not patched over in translation.

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
3. **Length / layout** — if a HARD/SOFT render site plus the target's expansion looks risky, **flag the key for `/check-fit`** and keep the faithful translation. Never silently truncate, invent abbreviations, or pre-shorten to dodge a guessed metric — fitting is `/check-fit`'s job.
4. **Glossary** — same source term → same target term everywhere; lock product/feature/brand names. **Polysemy guard:** if one source word appears with different functions (e.g. tab "Clean" = remove junk vs. "all clean" = state), detect it and decide deliberately whether to unify or split the term; record both.
5. **Tone/register** — reads native at the voice card's energy level, not dictionary-literal.
6. **Intent fidelity** — same meaning, scope, action; nothing added/softened/strengthened/re-scoped.

**3d. Self-critique:** each string must pass BOTH *"carries the designer's exact intent?"* AND
*"native & on-voice in-app?"*. If they conflict, keep fidelity and flag the tension.

---

## Overflow & layout fit → `/check-fit`

Fitting translated text to tight UI is a **separate concern owned by the view layer**, not the
translation step. When Stage 1/3 flags a key as overflow-risky, hand it to the **`/check-fit`**
workflow, which runs the full protocol (real budget from Compose/View constraints, font-fallback
metrics, an *optional last-resort* shorten-the-form step with dev approval, and escalation to
layout fixes like autosize / wider container). Here you only **translate faithfully and flag** —
never pre-shorten, abbreviate, or change meaning to make something fit.

---

## Glossary protocol
Build a term table (source → target → rationale); lock product/feature/brand names and
recurring UI nouns; reuse on every occurrence (no synonym drift); apply the polysemy guard;
output the table for reuse. A dev-provided glossary is authoritative.

## Anti-patterns
Word-by-word ignoring function; copying English Title Case; translating inside placeholders;
loose paraphrase of legal/security; "improving" the source; changing action/severity/scope;
sounding more expressive than the voice card; silent truncation; over-shortening to dodge a
font-metric guess; owning layout-fit in the translation step (that's `/check-fit`'s job);
adding or removing `<![CDATA[…]]>` around markup instead of preserving the source's form;
concatenating fragments instead of a parameterized resource; coining a new term for something
already translated elsewhere. (File-level anti-patterns — bare `%s`, unescaped `'`/`&`/`<`,
dumping translation notes into XML, auto-writing without a diff — are enforced by the
`android-xml` rule.)
