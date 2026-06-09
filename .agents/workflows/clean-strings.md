---
description: Audit a project's Android string resources and remove the unused ones safely. Scans every surface a key can be referenced from (code in all modules/source sets, all res XML, the manifest), classifies candidates by confidence, and deletes only what the dev confirms — cascading across every locale, then verifying with a build. Use when asked to "clean up strings", "find unused strings", "remove orphan keys", "dọn string", "audit resources" — i.e. a broad cleanup with NO specific key named. For deleting one named key, use a targeted delete instead.
---

# /clean-strings

Find and remove dead string resources **safely**. Default mode is a read-only audit; deletion
is a gated step. Core safety property: this workflow deletes **only keys with zero references
anywhere**, so removing them can't create a dangling `R.string.key` — it never edits code.

Push the heavy scanning into the **shell** and read only the distilled result into context
(read economy): a typical run costs a few k tokens and runs once, not per string.

---

## Phase 1 — Scan (shell-driven; the model reads only the final lists)

Build two sets and subtract them. Adjust module/source-set paths to the repo.

```bash
# 1) Defined keys (string / plurals / string-array), across base + every locale
grep -rhoE 'name="[^"]+"' $(find . -path '*/res/values*/strings.xml' -not -path '*/build/*') \
  | sed -E 's/name="(.*)"/\1/' | sort -u > /tmp/defined.txt

# 2) Referenced keys — code (ALL modules & source sets) + every res XML + manifest
{ grep -rhoE 'R\.(string|plurals|array)\.[A-Za-z0-9_]+' --include=*.kt --include=*.java . \
      | sed -E 's/.*\.//'
  grep -rhoE '@(string|plurals|array)/[A-Za-z0-9_]+' --include=*.xml . \
      | sed -E 's#.*/##'
} 2>/dev/null | sort -u > /tmp/referenced.txt

# 3) Candidates = defined − referenced (pure shell; no file read into context)
comm -23 /tmp/defined.txt /tmp/referenced.txt > /tmp/candidates.txt
wc -l /tmp/defined.txt /tmp/referenced.txt /tmp/candidates.txt
```

Only `/tmp/candidates.txt` (usually small) needs to enter context. Don't read whole resource
or code files.

## Phase 2 — Classify each candidate (🟢 / 🟡 / ⛔)

| Tier | Condition | Fate |
|---|---|---|
| 🟢 **SAFE** | 0 references anywhere, not dynamic, not recently added | eligible to delete |
| 🟡 **REVIEW** | referenced only in tests/another flavor; recently added (`git log`/uncommitted); is a `plurals`/`array` | dev decides |
| ⛔ **KEEP** | appears in manifest / reachable via `getIdentifier` / `translatable="false"` / public string of a library module | never touch |

## Safety brakes (apply before anything lands in 🟢)

```bash
grep -rl 'getIdentifier' --include=*.kt --include=*.java . 2>/dev/null   # dynamic key names?
grep -rl 'translatable="false"' $(find . -path '*/res/values/strings.xml') 2>/dev/null
git -C . log --since='14 days ago' -p -- '**/values/strings.xml' 2>/dev/null  # recently added keys
```

- **`getIdentifier` present** → loudly warn; keys whose names could be built at runtime are **not** SAFE (grep can't see a name spliced from a variable).
- **`translatable="false"`** → KEEP by default (usually used from manifest/build).
- **Recently added / uncommitted** keys → REVIEW, not SAFE (don't undo a dev's in-progress feature).
- Also report (don't auto-act): **duplicate values** (propose merge) and **keys missing from some `values-<lang>/`** (suggest a translation top-up).

## 🚦 GATE — Report, then the dev picks the delete set

Print the classified report (🟢/🟡/⛔ with the actual key names + counts). **Delete nothing**
until the dev confirms the final set (defaults to 🟢; the dev may add specific 🟡 keys). If
`getIdentifier` was found, say so explicitly so the dev weighs the risk.

## Phase 3 — Delete (the shared delete-engine)

For each confirmed key:
- Remove its element from **`res/values/strings.xml` AND every `res/values-*/strings.xml`** — never skip a locale, or you leave an orphan translation (`ExtraTranslation`).
- `<plurals>` / `<string-array>` → remove the **whole block** (all `<item>` children).
- **Element-aware** removal (from the opening tag through its closing tag, including CDATA / multi-line values); leave every other entry's formatting untouched so the diff stays minimal.
- Structural comments: **don't auto-delete**; if a section (`<!-- Onboarding -->`) becomes empty, flag it for the dev rather than removing it.
- Tooling: prefer `xmlstarlet ed -L -d "//string[@name='KEY']"` when available (XML-aware); otherwise an exact-match edit per entry (each `<string name="KEY">` is unique).
- Stage everything as **one commit** so the whole cleanup reverts in one step. Ensure the working tree is clean (or on a branch) first.

## Phase 4 — Verify (the final safety net)

```bash
./gradlew :app:processDebugResources :app:compileDebugKotlin :app:lintDebug
```

- 🟢 green → done. Report: `deleted N keys from M files`.
- 🔴 red → because only 0-reference keys were deleted, red means the **scan missed a reference**
  (e.g. a dynamic `getIdentifier` name). **Auto-revert the commit**, name the offending key, and
  move it to KEEP. Never leave the tree broken.

---

### Quick reference
```
/clean-strings → (optional: limit to a module)
  1 SCAN (shell): defined keys − referenced keys (code all modules/source sets + res XML + manifest) = candidates
  2 CLASSIFY 🟢 SAFE / 🟡 REVIEW / ⛔ KEEP
     brakes: getIdentifier → not SAFE; translatable=false → KEEP; recently-added → REVIEW
     also report: duplicate values, locales missing keys
  3 🚦 report → dev picks delete set (default 🟢)
  4 DELETE: element-aware, base + EVERY locale, plurals/array whole block, 1 commit
  5 VERIFY build+lint → green: done · red: auto-revert + flag missed ref
```
