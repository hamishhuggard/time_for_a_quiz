---
name: quiz-ascii-audit
description: Audit a "Time for a Quiz" deck CSV for questions that can't be answered correctly with plain ASCII keyboard input (no Unicode symbols, no copy-paste), and patch the Answers column so an ASCII alternative is accepted. Use when asked to check/fix a deck for ASCII-typability, when a student reports a "correct" answer being marked wrong, or after adding/editing questions with symbols, units, or non-English names.
---

# Quiz ASCII audit

This repo (`time_for_a_quiz`) is a single-file quiz app (`index.html`) driven
by deck CSVs in `decks/`. Answers are graded by exact/near-exact string or
numeric match against a pipe-delimited `Answers` list — there is **no Unicode
normalization**, so if every accepted alternative for a question contains a
symbol a student can't type on a plain keyboard (Ω, ×, °, superscript digits,
√, π, non-ASCII accented letters with no plain-ASCII alternative spelling,
etc.), that question is effectively unanswerable. This skill is the repeatable
process for finding and fixing those gaps.

## 1. Serve the app and start a quiz

```
cd /Users/hamish/Desktop/time_for_a_quiz
python3 -m http.server 8123   # pick a free port -- 8000 is often already in use by other local projects
```

Open `http://localhost:8123/index.html` in Chrome (via claude-in-chrome tools:
load `mcp__claude-in-chrome__tabs_context_mcp`, `navigate`, `computer`,
`read_page`, `tabs_create_mcp`, `find`, `get_page_text` via ToolSearch first).
**Never use `file://`** — the deck CSV is loaded via `fetch()`, which is
blocked on `file://`, and you'll silently see nothing load.

On the start screen: pick the deck, set level mode to "All Levels Randomly"
(broadest coverage), leave Danger Mode unchecked (it kills you on a timer —
not useful for methodical testing), and give yourself a generous main time
limit (15 min is plenty for a ~30-question deck). Click "Set Up & Start" then
"Start Quiz" on the confirmation screen.

Answering loop: find the answer `<input>` via `find` ("answer input text
field"), click it, type the ASCII answer, press Enter (submits), press Enter
again (advances — the Next button grabs focus after a submit). Use
`get_page_text` after each pair of Enters to read the next question and any
`✅ Correct` / `⚠️ Close` / `❌ Incorrect` feedback. Batch click+type+Enter+Enter+get_page_text
into one `browser_batch` call per question to move fast.

**Cache gotcha**: if you edit a deck CSV mid-session and want to re-test, the
browser may serve a cached copy of the CSV (`python3 -m http.server` doesn't
send cache-busting headers). Force a fresh fetch by reloading with a
cache-busting query string appended to the CSV path, e.g. navigate to
`http://localhost:8123/index.html?csv=decks%2F<deck>.csv%3Fv%3D2` (the `%3F`/`%3D`
are an escaped `?v=2`) rather than just re-clicking Start.

## 2. How grading works (`index.html`)

- `submitAnswer()` (~line 975) does `document.getElementById('answer-input').value.trim().toLowerCase()`
  then calls `processResult(inputVal, false)`.
- Deck answers are lowercased once at load time (`instantiateQuestion`, ~line 584: `.toLowerCase()`).
- `processResult()` (~line 887) checks, per alternative `ans` in `currentQuestionObj.a`:
  1. **Exact string match**: `inputVal === ans`.
  2. **Numeric match**: both `inputVal` and `ans` parsed with `parseScientific()`
     (~line 527) — handles `x10^`, `*10^`, `e` notation, strips whitespace/commas
     — matched within 0.1% relative tolerance. `parseScientific` does **not**
     understand or strip Unicode (Ω, ×, °, superscript/subscript digits are just
     left in the string and fail `Number(...)` parsing).
  3. **"Close" flag** via `getEditDist()` (~line 873, plain Levenshtein):
     edit distance 1 (or 2 if the alternative is >5 chars) sets `close = true`.
     **`close` is NOT a pass** — it only changes the feedback message
     (⚠️ "Close!" vs ❌ "Incorrect"), the question is still marked wrong, pushed
     back into the queue, and no points are awarded.
- Net effect: if every alternative in `Answers` requires a Unicode character
  or an unusual spelling, and the ASCII-typable phrasing a real student would
  use differs by more than 1-2 edit-distance characters, the question is
  unanswerable by keyboard.

## 3. The failure signature to search for

Grep the deck CSV for the columns and read each `Answers` list (pipe `|`
delimited). Flag a row if:
- **Every** alternative contains a Unicode symbol (Ω, μ, Δ, ×, °, √, π,
  superscript/subscript digits, accented letters like é/è) with no plain-ASCII
  alternative spelling present (e.g. only `Ω` with no `ohm`).
- Alternatives cover only one exact phrasing/word order and a differently-
  ordered-but-equally-correct ASCII phrasing isn't listed (e.g. "energy and
  electric field strength" vs "electric field strength and energy").
- A name/unit has a hyphenated form listed but not the space-separated form
  a student would actually type (e.g. `Andre-Marie Ampere` present but
  `Andre Marie Ampere` absent — edit distance from the space version to the
  hyphenated one is 1, which is only "close", not correct).
- A unit has a singular form listed but not the natural plural/colloquial form
  (e.g. `amp` present but not `amps`; check singular/plural and formal/informal
  both ways).
- US vs UK spelling gaps (`metre` present, `meter` absent).

Don't flag rows that already have a plain-ASCII alternative (e.g. `ohm|ohm
(Ω)|Ω` is fine because `ohm` alone is ASCII and matches exactly) — verify live
in the browser before treating something as a bug, since it's easy to
misjudge what counts as "close enough" for the edit-distance fallback.

## 4. The fix convention

Edit the deck CSV directly (never `index.html`'s grading logic — the fix is
always broadening `Answers`, not touching the grading code, unless you find
an actual app crash/bug, which should be flagged separately, not silently
patched). Append new plain-ASCII alternatives to the existing pipe-delimited
`Answers` list, keeping all existing alternatives and the underlying correct
value unchanged. Follow the deck's existing style, e.g.:

```
What is the unit for resistance?,ohm|ohm (Ω)|Ω,,1
What is the unit for current?,ampere (A)|ampere|amperes|amp|amps|A,,1
The ampere is named after which scientist?,Andre-Marie Ampere|Andre Marie Ampere|Ampere|André-Marie Ampère|Ampère,...
```

Don't touch `Question`, `Explanation`, or `Level` columns unless they're
clearly wrong — this skill is scoped to the `Answers` column.

## 5. Re-test after every fix

After editing, reload with a cache-busted `?csv=` URL (see §1 cache gotcha),
navigate through the deck (spaced-repetition requeueing means a missed
question comes back ~3 questions later — or just restart a fresh session and
answer everything correctly except the target question, then retype your
previously-failing ASCII answer) and confirm you now get `✅ Correct!` instead
of `⚠️ Close!` / `❌ Incorrect`.

## 6. Adding a new deck

The `DECKS` array (`index.html` ~line 253) lists every deck the start screen
offers:

```js
const DECKS = [
    { name: "Electromagnetism: units, quantities & symbols", file: "decks/electromagnetism_units_quantities_symbols.csv" }
];
```

If a new CSV is added to `decks/`, add an entry here (`name` = display label,
`file` = relative path) so it shows up in "Choose a deck". This skill's
workflow (serve, play, grep, patch, retest) applies unchanged to any deck in
the array — just make sure to select the deck under test in the "Choose a
deck" dropdown before hitting "Set Up & Start".

## Suggested end-to-end workflow

1. Serve the app, start a quiz on the target deck with broad settings
   (all levels, no danger mode, generous timer).
2. Play through methodically, answering with the ASCII input a real student
   would type. Prioritize rows with symbols/units in the source CSV.
3. On any incorrect/close result, first double check the actual fact so you
   don't mistake your own mistake for a bug, then grep the deck CSV for the
   question text to find the exact row.
4. Confirm the row's `Answers` list genuinely lacks an ASCII alternative.
5. Patch the CSV: append the ASCII alternative(s), pipe-delimited, alongside
   existing alternatives.
6. Reload with a cache-busted `?csv=` URL and retest the same input —
   confirm `✅ Correct!`.
7. Repeat until you've covered every row in the deck (small decks: aim for
   every row, not a sample).
