---
description: Localize Android Kotlin/Compose strings into a target language from inside the repo. Runs scope → voice → code-read → translate → diff → write, with confirm-gates, preserving the designer's intent and the real layout limits read from Compose. Use when asked to "localize the app", "translate the strings", "dịch string", add a language, or populate values-xx/strings.xml.
---

# /localize-app

Guided run of the **android-string-translator** skill (the expertise) with the **android-xml**
rule enforcing file mechanics. This file is the *sequence and the gates* — pull the detail from
the skill at each step; don't duplicate it here.

Ask up front if unknown: **target language(s)?** Then run the stages in order. Two gates are
mandatory — **never auto-write**.

---

## Step 1 — Scope: what to translate

Don't assume "the whole file." Resolve the exact target set:

- **Partial / incremental (most common):** only the new or changed keys. Resolve from whatever the dev gives:
  - explicit key list → trust it
  - a diff/PR/commit: `git diff --name-only` then inspect changed `strings.xml` entries, or `git diff -- '**/values/strings.xml'`
  - keys present in `res/values/strings.xml` but missing from `res/values-<lang>/strings.xml`
  - a named feature area
- **Full file:** the entire `res/values/strings.xml`.

If the set is ambiguous, **confirm the exact keys before translating.** In partial scope, touch
only the target keys — never re-translate or "fix" already-shipped translations.

## Step 2 — Voice card (detect → confirm → lock)

Run **Stage 0** of the skill: infer the voice card from existing strings + Compose `Text`,
present it, and ask once **keep or shift**.

🚦 **GATE 1:** wait for the dev to confirm/lock the voice card before translating. A
dev-supplied card is used verbatim. The locked card is the personality ceiling for every string.

## Step 3 — Trace & read the code (constraints)

Run **Stage 1** of the skill with read economy — narrow, not repo-wide:

```
grep -rn "R.string.<key>\|stringResource(R.string.<key>" app/src
```

For each in-scope key: open only the render-site composable grep returns, read ~20 lines around
the `Text`, and extract budget + tier (HARD/SOFT/FLEXIBLE), `maxLines`/`softWrap`/autosize,
`width`/`weight`, container widget, and any config-qualified variants (localize for the
tightest). Note Kotlin fragment-assembly (`a + count + b`) → flag for a parameterized resource
instead of translating pieces. Check font coverage for the target script (Step A2) if it's CJK/KR/Thai/etc.

## Step 4 — Translate by intent + validate

Run **Stages 2–3** of the skill: classify each string, build the target-language profile once,
translate with intent/scope/content fixed and register tuned to the voice card. Then run the
hard checks — variables repositioned, plurals with the target's forms, length vs. budget ×
expansion, glossary + polysemy, register, fidelity — and the BOTH-must-pass self-critique. For
anything at risk of overflow, run the Overflow protocol (Step B shorten-form; register-weak or
won't-fit → Step C escalate to the dev).

## Step 5 — Prepare the file, preview, confirm, write

Flow: **prepare → preview → confirm → write.** The **android-xml** rule governs the file
mechanics (locale folder + BCP47 qualifier, escaping, positional args, plurals syntax,
`translatable="false"`, CDATA, key order).

1. Generate the target `res/values-<qualifier>/strings.xml` (create the folder if missing). In partial scope, insert only the new keys without reordering existing entries.
2. Show the dev a **diff** limited to the added/changed keys.

🚦 **GATE 2:** write only after the dev confirms the diff. Never write silently.

## Step 6 — Report alongside the file (chat/PR, not in XML)

- Decision notes: `key → choice + 1-line why`, only where non-obvious.
- Layout & font-fallback warnings with shorter alternatives / recommended fixes.
- Glossary table (for reuse).
- Flags: ambiguous source, fragment assembly, non-plural-safe or bad source strings to fix in code (e.g. multi-arg strings using bare `%s` that must become positional `%1$s` **in the source** before any RTL/CJK locale is added).

Keep it terse; group flags for large batches.

---

### Quick reference
```
/localize-app → target language?
  1 scope: new/changed keys only, or full file? (git diff / list / missing in values-xx) — confirm if ambiguous
  2 voice: detect from strings+Compose → confirm keep/shift → 🚦 lock
  3 trace each key (grep) → read Compose: budget, tier, wrap, autosize, font coverage
  4 classify → target-lang profile → translate (intent fixed, register to voice)
     checks: vars ✓ plurals ✓ length ✓ glossary+polysemy ✓ register ✓ fidelity ✓
     overflow? Step B shorten-form; weak/won't-fit → Step C escalate
  5 prepare values-xx/strings.xml (rule: escape, positional args, plurals) → diff → 🚦 confirm → write
  6 notes + warnings + glossary + flags → chat/PR, never into the XML
```
