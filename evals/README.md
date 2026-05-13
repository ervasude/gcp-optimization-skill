# Evaluations

Test cases for the `gcp-optimization` skill. These are prompt/expectation pairs used to verify the skill triggers the right reference section and surfaces the right concepts.

## Structure

Each eval has:

- `id` — stable identifier (e.g. `bq-001-partition-pruning-basic`)
- `prompt` — the user message to send
- `expected_section` — the reference file + anchor the answer should draw from (null if out of scope)
- `expected_topics` — keywords or phrases that should appear in the response
- `category` — `diagnostic`, `factual`, `conceptual`, `myth-busting`, `decision`, `best-practice`, `preventive`, `scope-test`, `edge-case`

## Categories

- **diagnostic** — "something is wrong, what's the cause?"
- **factual** — "is X allowed/supported?"
- **conceptual** — "why does X work this way?"
- **myth-busting** — common misconceptions (e.g. "does LIMIT reduce cost?")
- **decision** — "should I use X or Y?"
- **best-practice** — "how do I do X correctly?"
- **preventive** — "how do I stop X from happening?"
- **scope-test** — out-of-scope prompts; the skill should not engage
- **edge-case** — prompts that span multiple references or hit incomplete sections

## Running evals manually

1. Open Claude.ai (or any agent) with the skill loaded.
2. Paste a prompt from `evals.json`.
3. Check that the response references the expected section and covers the expected topics.
4. Mark as pass / fail in your tracking sheet.

## Results

Current status (manually tested on Claude.ai with skill enabled):

| ID | Status  |
|---|---------|
| bq-001 | PASS    |
| bq-002 | PASS    |
| bq-003 | PASS    |
| bq-004 | PASS    |
| bq-005 | PASS    |
| bq-006 | PASS    |
| bq-007 | PASS    |
| bq-008 | PASS    |
| bq-009 | PASS    |
| gcs-001 | pending |
| gcs-002 | pending |
| gcs-003 | pending |
| gcs-004 | pending |
| scope-001 | pending |
| scope-002 | pending |


## Future work
- Automate eval execution via Claude API
- Add evals for `compute.md` and `data-pipelines.md` once those sections are filled
- Track pass rates across skill versions to catch regressions