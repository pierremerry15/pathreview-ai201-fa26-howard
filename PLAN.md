## Solution plan

**Issue:** [#53 — PII scrubber fails to redact parenthesized US phone numbers](https://github.com/codepath/pathreview-ai201-fa26-howard/issues/53)

**Branch:** `fix/53-pii-scrubber-parenthesized-phone`

**Reproduction commit:** [`ce2dde8`](https://github.com/pierremerry15/pathreview-ai201-fa26-howard/commit/ce2dde861dd3777b3b0f810404c8d94f3a4f2ef6)

---

### Understand

**Expected:** every common way of writing a US phone number is replaced with `[REDACTED]`
in full by `PIIScrubber.scrub()`, and `PIIScrubber.detect()` reports it as a `phone_us`
hit whose `value` is the whole number.

**Actual:** reproduced on my machine with the issue's own snippet and with the
per-format tests in `ce2dde8`:

| Input | `scrub()` today | `detect()` today |
|---|---|---|
| `555-123-4567` | `[REDACTED]` | `phone_us` ✓ |
| `555.123.4567` | `[REDACTED]` | `phone_us` ✓ |
| `5551234567` | `[REDACTED]` | `phone_us` ✓ |
| `1-555-123-4567` | `[REDACTED]` | `phone_us` ✓ |
| `(555) 123-4567` | unchanged | nothing |
| `+1 555 123 4567` | unchanged | nothing |
| `+1 (555) 123-4567` | unchanged | nothing |
| `555 123 4567` | unchanged | nothing |
| `(555)123-4567` | `([REDACTED]`, a stray paren is left | `555)123-4567`, the wrong span |

**Root cause: two defects in the one `phone_us` regex.**

1. **The separator class is `[-.]`.** It allows a hyphen, a dot or nothing, but never a
   space. Every format that puts a space between number groups fails, and
   `(555) 123-4567` is the most common of them. This is what the issue reports.
2. **The leading `\b` cannot sit in front of `(`.** `\b` needs a word character on one side.
   In `at (555)` there is a space before `(` and `(` itself, so there is no boundary there.
   The match can only begin at `555`. The optional `\(?` is never used, and the
   opening paren is left outside the match. Defect 1 hides this for `(555) 123-4567`, but
   it shows up on its own in `(555)123-4567`. The same thing happens with `+`, so even with
   whitespace allowed, `+1 555 123 4567` would come out as `+[REDACTED]`.

Both symptoms in the issue, from `scrub()` and from `detect()`, have the same cause: both
methods loop over the same `PII_PATTERNS` dict. Fixing the pattern fixes both. Neither
method body needs to change.

**A second root cause I found while reproducing:** one of the five tests marked for #53,
`test_mixed_pii_and_text`, is **not** failing because of phone numbers at all. Its phone is
the dashed `555-123-4567`, which already redacts. It fails because the `street_address`
pattern matches `5 years developing Python appl` in `"for 5 years developing Python
applications"`. The pattern is case-insensitive and has no word boundary around the suffix,
so the `pl` in "applications" counts as the `Pl` (Place) suffix. The match takes the word
"Python" with it, and `assert "Python" in scrubbed` fails. Fixing the phone pattern alone will
leave this test failing. If I deleted its marker on that basis, CI would turn red for a
bug I hadn't fixed.

### Map

| File | What's there | What I'll change |
|---|---|---|
| `safety/pii_scrubber.py` → `PIIScrubber.PII_PATTERNS["phone_us"]` | The broken regex | Rewrite it (sub-task 1) |
| `safety/pii_scrubber.py` → `PIIScrubber.PII_PATTERNS["street_address"]` | The over-matching regex | Add word boundaries around the suffix group (sub-task 3) |
| `tests/unit/test_pii_scrubber.py` | 5 original `xfail(strict=True)` tests for #53, plus my 10 per-format xfail cases and 5 guard cases from `ce2dde8` | Remove every #53 marker, then add regression cases |
| `JOURNAL.md` | Running log | Unit 9 entry |

**Read and confirmed, no change needed:**
- `PIIScrubber.scrub()` / `detect()`: both iterate `PII_PATTERNS`. `detect()` reports
  `match.group()`, `match.start()` and `match.end()` and never uses capture groups 1–3, so I
  can drop or restructure the groups without changing its output.
- `pyproject.toml`: `safety/pii_scrubber.py` has a per-file `E501` ignore for the long regex
  lines, and there is no mypy override for `safety`. No suppression is tied to #53, so there
  is nothing to remove there.
- `safety/monitoring.py`: logs a `pii_detected` event, but it does not import `PIIScrubber`.

**Something I found while tracing callers:** a search across `api/`, `agent/`, `core/`,
`ingestion/` and `rag/` turns up **no production code that imports or calls
`PIIScrubber`**. Only the unit tests use it. `docs/ARCHITECTURE.md` describes it as the last
step of the safety middleware chain. In this snapshot it isn't connected to the pipeline,
which issue #31 ("integration tests for the full safety middleware chain") looks
related to. That keeps my change's blast radius small. It also means I can verify the fix only
through unit tests, not through the running app. Connecting the scrubber to the pipeline is
out of scope for #53.

### Plan

1. **Rewrite `phone_us`** in `safety/pii_scrubber.py` to:

   ```
   (?<![\w+])(?:\+?1[-.\s]?)?(?:\(\d{3}\)|\d{3})[-.\s]?\d{3}[-.\s]?\d{4}(?!\w)
   ```

   - `[-.\s]?` allows **at most one** hyphen, dot or whitespace character between
     groups. I'm not using `\s*` on purpose, so that a number can't stretch across
     arbitrary gaps.
   - `(?:\(\d{3}\)|\d{3})` accepts the area code either in a **balanced** pair of parens or
     bare. The old `\(?…\)?` accepted half-open forms like `(555 123` and `555) 123`.
   - `(?<![\w+])` / `(?!\w)` replace both `\b`s. The match can now start on `(` or `+`, so
     no stray characters are left behind. It still can't start or end inside a longer run
     of digits or letters.
   - Capture groups are dropped, which is safe because `detect()` never reads them.

   I've already prototyped this in a scratch copy. All 9 formats in the table above redact
   fully, and all 5 guard cases still pass.

2. **Remove the #53 `xfail` markers that the phone fix satisfies.** These are the four
   tests the issue names (`test_us_phone_number_redaction`, `test_us_phone_formats`,
   `test_detect_phone_pii`, `test_phone_at_start_of_text`) and the 10 per-format cases from
   `ce2dde8`. I'll remove the `_ISSUE_53` mark object along with them. Then I'll run
   `make test-unit` and confirm there are no `XPASS(strict)` failures.

3. **Fix the `test_mixed_pii_and_text` failure.** Add `\b` before and after the suffix
   alternation in `street_address`, so a suffix has to be a whole word. In the prototype,
   `"5 years developing Python applications."` comes through untouched, while
   `123 Main Street` and `42 Elm Rd` still redact. Then remove that test's marker.
   **I'll check with a maintainer in Slack first** (see Risks). If they want #53 to stay
   phone-only, I'll leave this marker on with its reason corrected to name the real cause,
   and open a separate issue for `street_address`.

4. **Add regression tests** in `tests/unit/test_pii_scrubber.py`:
   - phone inside surrounding punctuation: `"(call (555) 123-4567)"` and `"Phone:(555) 123-4567,"`
   - two numbers in one string, both redacted, and `detect()` returning two `phone_us` hits
     with correct `start`/`end`
   - a `street_address` negative case, `"developing Python applications"` stays intact,
     and positive cases, so the boundary fix is pinned down independently of the mixed test

5. **Run the full CI set locally and write it up.** `make check` (ruff, black, mypy),
   `make test-unit`, and `cd frontend && npm test -- --run`. Record the before/after
   pass/xfail counts in `JOURNAL.md`. The expected change is exactly the #53 cases moving
   from xfailed to passed, plus the new tests. Then open the PR with `Fixes #53` and a note
   explaining the `street_address` change.

### Inputs & outputs

- **Input:** any `str`, typically resume text, a free-text profile field, or generated
  feedback, passed to `PIIScrubber.scrub(text)` or `PIIScrubber.detect(text)`.
- **Output of `scrub()`:** the same string with each US phone number in any common format
  replaced by exactly one `[REDACTED]`, with no leftover `(`, `)` or `+`. All other text
  stays byte-for-byte the same.
- **Output of `detect()`:** a list of `{"type", "value", "start", "end"}` dicts. A phone
  produces one `phone_us` entry with `value` equal to the full number as written and
  `text[start:end] == value`.
- **Unchanged:** both methods' signatures, the `[REDACTED]` token, the order of the patterns,
  and the email, SSN and international patterns.

### Risks & unknowns

- **Over-redaction from widening the separator.** This is the main risk. Allowing a space
  means any `3 digits, space, 3 digits, space, 4 digits` run now matches. The prototype
  turns `listed 100 200 3000 items` into `listed [REDACTED] items`. I think this is
  acceptable, because for a PII scrubber redacting too much is safer than leaking. I'll
  document it as a known trade-off in the PR rather than hide it. The guard tests (ISO date,
  ZIP+4, 14-digit identifier, version number, 7-digit local number) pin down what must
  not change.
- **Scope of the `street_address` fix.** The test is labelled #53, and `CONTRIBUTING.md`
  says a fix should remove the marker from *every* test covering the issue. But the cause is
  a different pattern, and `CONTRIBUTING.md` also says not to fold unrelated fixes into a
  bug PR. I need a maintainer to decide between fixing it here as a separate commit and
  splitting it out. Either way, I won't delete that marker without the test actually passing.
- **Pattern order inside `scrub()`.** The patterns run one after another, each on the output
  of the previous one. `phone_us` runs before `phone_intl` and `ssn`. I checked that
  `+1 555 123 4567` is now consumed entirely by `phone_us`, and that `123-45-6789` (3-2-4)
  still reaches `ssn`, because it can't satisfy the 3-3-4 phone shape. `street_address`
  needs a leading digit run, so it can't start on a `[REDACTED]` token left behind by an
  earlier pattern (`Phone: [REDACTED] Lane info` stays as is). What I haven't tested yet
  is every pattern pair. I'll add a test that combines all the PII types in one string
  before calling this done.
- **International numbers stay partly redacted.** `+44 20 7946 0958` currently becomes
  `[REDACTED] 20 7946 0958`, because `phone_intl` has the same `[-.]` separator. That's an
  existing gap, not #53. The current test accepts it, so I'll leave it out of scope and
  record it as a possible follow-up issue.
- **The scrubber isn't connected to the app** (see Map). I can't demo the fix through the
  UI. Verification is unit tests only.
- **My local runs differ from CI.** Python here is 3.13, and CI targets 3.11. The regex
  features I'm using (fixed-width lookbehind, `\s`, `\d`) behave the same in both, but CI is
  the final word.

### Edge cases

| Case | Expected after the fix |
|---|---|
| `(555) 123-4567`, `(555)123-4567`, `+1 (555) 123-4567` | Fully redacted, no stray `(` or `+` |
| `+1 555 123 4567`, `555 123 4567` | Fully redacted |
| Phone at the very start / very end of the text | Redacted (lookarounds handle string edges) |
| Phone wrapped in punctuation: `(call (555) 123-4567)` | Number redacted, outer parens kept |
| Two phones in one string | Both redacted, two `detect()` hits with correct offsets |
| Unbalanced paren: `(555 123-4567` | `555 123-4567` redacted, the lone `(` is left |
| Phone with double spaces: `555  123  4567` | **Not** redacted, since only one separator character is allowed. Accepted limitation, to be noted in the PR |
| ISO date `2026-11-11`, ZIP+4 `20059-1234` | Unchanged |
| 14-digit identifier, version `1.2.3`, 7-digit `555-1234` | Unchanged |
| SSN `123-45-6789` | Still redacted, by `ssn` |
| Empty string / whitespace only | Returned unchanged; `detect()` returns `[]` |
| Text that's already scrubbed | `scrub(scrub(x)) == scrub(x)` (idempotent) |
| `5 years developing Python applications` | Unchanged once the `street_address` boundary fix is in |
