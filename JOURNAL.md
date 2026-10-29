# PathReview Contribution Journal

Running record of my Module 3 contribution to PathReview.

---

## Unit 7 — Issue selection

**Issue link:** https://github.com/codepath/pathreview-ai201-fa26-howard/issues/53

**Issue title:** PII scrubber fails to redact parenthesized US phone numbers

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**

PathReview runs user-supplied text through `PIIScrubber` in the safety layer before it
reaches the language model or gets persisted, so the scrubber is the component responsible
for making sure personal details never leave the system in the clear. Its `phone_us` pattern
permits only a hyphen or a dot between the area code, exchange, and line number, which means
any number written with a space separator never matches — including `(555) 123-4567`, the
most common way a US phone number is actually written, and `+1 555 123 4567`. The practical
consequence is that `scrub()` hands those numbers back untouched and `detect()` reports no
PII for them at all, so a resume carrying a parenthesized phone number is forwarded to the
model and stored with the number intact, while the same number written with dashes is
correctly redacted. A successful fix widens the separator so every common US format redacts
consistently, without loosening the pattern so far that it begins swallowing unrelated digit
runs such as dates, ZIP+4, or identifiers. The work is contained to `safety/pii_scrubber.py`
and is already covered by five tests in `tests/unit/test_pii_scrubber.py` that currently
carry `@pytest.mark.xfail(strict=True)`.

**Branch name:** `fix/53-pii-scrubber-parenthesized-phone`

**Setup confirmation:** [ ] App runs locally at localhost:5173

**Cohort ledger:** [ ] Issue added to cohort ledger

---

### "Is this right for me?" — selection reasoning

**Scope is genuinely small, and I confirmed that by reading rather than assuming.**
The defect lives in a single dictionary entry — one regex string in `PII_PATTERNS` in
`safety/pii_scrubber.py`. Both public methods, `scrub()` and `detect()`, iterate that same
dictionary, so correcting the one pattern fixes both reported symptoms without touching
either method body.

**I reproduced the failure before committing to the issue.** Running the existing pattern
against the five formats in the test fixture isolates the break exactly:

| Input | Result |
|---|---|
| `555-123-4567` | redacted |
| `(555) 123-4567` | **not redacted** |
| `555.123.4567` | redacted |
| `+1 555 123 4567` | **not redacted** |
| `5551234567` | redacted |

The two failures are precisely the two formats that use a space as a separator. The pattern's
separator class is `[-.]`, which admits a hyphen or a dot and no whitespace, so the match
consumes the area code, accepts the closing paren, then meets a space where it requires
`[-.]` or nothing and fails before reaching the exchange digits. That table is what moved me
from "this looks like a regex problem" to a specific named cause.

**Verification is already built for me.** Five tests covering this issue are marked
`xfail(strict=True)`. Because the marker is strict, a correct fix flips them from xfail to
XPASS and turns CI red until the markers are deleted — per `docs/CONTRIBUTING.md`, removing
them is part of the fix, not an afterthought. That gives me an unambiguous definition of done
and makes it hard to claim success without it.

**The risk I need to manage is over-redaction, not under-redaction.** Widening a separator
class is the kind of change that quietly starts matching things it shouldn't. `PII_PATTERNS`
also holds `ssn` and `phone_intl` entries, and the patterns are applied in sequence over the
same text, so a looser phone pattern could consume digits another pattern expects or redact
ordinary numbers in prose. My plan for Unit 8 is to decide deliberately how permissive the
separator should be — a single optional whitespace character rather than `\s*` — and to add
negative cases proving that dates, ZIP+4, and long identifiers are still left alone.

**Why not something larger.** This is my first contribution to a codebase of this size, and
the course is explicit that there is no extra credit for a harder tier. A Tier 1 issue with a
reproducible failure, a contained blast radius, and prepared tests lets me spend my effort on
the parts that are actually graded — contribution standards, process documentation, and a
clean PR — instead of spending the four weeks still trying to understand the problem.

**Open questions to resolve during implementation.** Whether `detect()` depends on the
pattern's three capture groups for reporting offsets, and whether `phone_intl` already
matches `+1 555 123 4567` through a different route, which would change what the fix for that
particular format needs to be.

---

### Setup status

Backend and frontend dependency installation and the `docker compose` services are not yet
running on this machine; Docker is not installed locally. Reproduction above was done with
the standard library against the pattern in isolation, so it does not depend on the full
stack. The `localhost:5173` checkbox stays unchecked until the app genuinely serves.
