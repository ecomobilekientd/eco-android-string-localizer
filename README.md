# Android String Localizer

A model-agnostic agent framework for localizing an **Android Kotlin/Compose** app's
user-facing strings from inside its repository. The agent reads the Compose UI to translate by
**intent and tone** — not word-by-word — while respecting the real layout limits, font
coverage, and on-screen context it finds in the code.

Built for [Google Antigravity](https://antigravity.google) (`.agents/` rules · skills ·
workflows), so it composes with the agent automatically instead of being one giant prompt.

## What it does

- **Fidelity first.** Preserves the designer's exact meaning, intent, and scope. The only lever
  it touches is tone/register/word energy — never the content, the CTA's action, or the claim.
- **Reads the code, not just the strings.** Traces each `R.string.<key>` to its `Text`, derives
  the layout budget (HARD/SOFT/FLEXIBLE), `maxLines`, autosize, width/weight, and config
  variants — then localizes for the tightest render site.
- **Catches what breaks builds.** Positional format args (`%1$s` not bare `%s`), XML escaping,
  per-language plural forms, `translatable="false"`, CDATA, correct `values-<bcp47>` folders.
- **Catches what breaks layout silently.** Font fallback for CJK/KR/Thai when the app font
  lacks glyphs (metrics shift width *and* line-height), and a register guard that escalates a
  cramped HARD element to a design decision rather than accepting a fitting-but-wrong word.
- **Safe by default.** Scope-aware (only the new/changed keys in partial runs), diff-and-confirm
  before any write, notes/flags to chat or the PR — never into `strings.xml`.

## Structure

The single skill is split into three Antigravity primitives by **progressive disclosure** —
each piece loads only when it's relevant, keeping the always-on footprint small and avoiding
context rot:

| File | Type | Loads when | Owns |
|---|---|---|---|
| [`.agents/rules/android-xml.md`](.agents/rules/android-xml.md) | Rule (`glob: **/strings.xml`) | a `strings.xml` is in context | Mechanical XML correctness — escaping, positional args, plurals syntax, locale folders. Small on purpose. |
| [`.agents/skills/android-string-translator/SKILL.md`](.agents/skills/android-string-translator/SKILL.md) | Skill (agent-triggered) | the agent detects a localization task | The expertise — fidelity, voice card, reading Compose for constraints, classification, overflow & font-fallback protocol, glossary. |
| [`.agents/workflows/localize-app.md`](.agents/workflows/localize-app.md) | Workflow (`/localize-app`) | the dev types `/localize-app` | The ordered procedure with confirm-gates and the git/grep commands. |

## Install

Antigravity reads `.agents/` from the workspace (or git) root. To use this in an Android repo,
copy the `.agents/` tree into that repo's root:

```bash
cp -r .agents /path/to/your-android-app/
```

(Skills and rules also work from the global scope `~/.gemini/antigravity/` if you want them
across all projects.)

## Usage

Inside the Android repo, just ask — e.g. *"localize the strings I added for the paywall into
Japanese"* or *"add German and Korean"* — and the skill equips automatically. For a guided,
gated run type:

```
/localize-app
```

It will resolve scope, confirm the voice card, read the Compose constraints, translate by
intent, then show a diff and wait for your confirmation before writing.

## Source

The three files are split from a single eval'd skill (`localize_string_v2`). The split is for
progressive disclosure, not token savings: keep the always-on rule small, equip the skill only
on a localization task, and run the procedure only on `/localize-app`.

## Validation status

The underlying logic was self-evaluated across mock and real files:

- **Overflow / tone / fidelity / read-economy / XML escaping** — pass; two early defects
  (register drift, polysemy) found and fixed.
- **Font fallback + multi-language** (de/ja/ko/zh/th/ar) — pass on structure: Arabic 6-form
  plurals, Thai vertical clipping, Arabic RTL/reorder all caught.
- **Real file — V9Compress (355 strings)** — caught the blocker: 15 multi-arg strings using
  **unnumbered** `%s` that must be converted to positional `%1$s %2$s` *in the source* before
  any CJK/Thai/Arabic locale is added; plus duplicate keys, `translatable="false"`, brand, CDATA.

**Known limit:** evaluation was self-run, so it proves the logic is covered, not that every
agent executes it perfectly. Sentence naturalness, Thai word-breaking, and Arabic bidi still
need native QA on-device.
