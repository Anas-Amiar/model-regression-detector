# Model Regression Detection System — the pitch

*A 2-minute walkthrough for presenting this project in an interview.*

## The 30-second version

"Most teams change an LLM prompt the same way they'd fix a typo — push it, see if
anyone complains. There's no equivalent of a unit test suite for 'does this prompt
still work correctly.' I built one: a CI pipeline that runs an AI feature against a
hand-verified golden dataset on every pull request, flags any drop in accuracy, and
blocks the merge if the regression is severe enough — the same way a failing test
suite blocks a bad code change."

## The problem, in plain terms

Imagine a customer support tool that uses an LLM to classify incoming emails as
billing, technical, account, or general. Someone tweaks the system prompt to fix one
edge case — and accidentally breaks classification for a different category nobody
tested. Without an automated check, that regression ships straight to production and
the team finds out from angry customers, not from CI.

## The idea

Treat prompt changes like code changes. Before any prompt or model change merges,
run it against a fixed set of test cases with known-correct answers, score it, and
compare the score to the last known-good baseline. If the drop is small, warn. If
the drop is severe, fail the build — exactly like a broken unit test would.

## How I built it (in order, and why that order)

1. **The feature under test** (`classifier.py`) — an email classifier with two modes:
   a mock mode (simple keyword rules, no API key needed) and a real mode (calls
   OpenAI using a versioned prompt). Built mock-first so the entire pipeline could be
   built and demoed without any API key.

2. **The versioned prompt** (`prompts/v1.yaml`) — the system prompt and few-shot
   examples live in a YAML file, not hardcoded in Python. That makes prompt changes
   show up as a clean git diff, reviewable like any other code change.

3. **The golden dataset** (`golden_dataset/v1.json`) — 10 hand-written test emails
   with verified-correct answers, spanning easy cases (clear billing/technical/account
   emails), medium cases (ambiguous wording), and hard edge cases (a one-word email,
   a non-English email, sarcasm, an email that spans two categories). Written by hand,
   deliberately never LLM-generated — the whole point is that the answer key is real
   ground truth, not the model grading its own homework.

4. **The eval engine** (`eval_engine/runner.py`) — runs every test case through the
   classifier and scores it: exact match on category, word-overlap heuristic on
   summary quality.

5. **The baseline + diff logic** (`eval_engine/history.py`, `eval_engine/diff.py`) —
   compares the current run's pass rate against a committed `baseline.json`, and
   classifies the result as ok / warning (3%+ drop) / critical (8%+ drop).

6. **Reporting + alerting** (`eval_engine/report.py`, `eval_engine/alerts.py`) —
   generates an HTML scorecard for every run, and sends a Slack alert on
   warning/critical (or prints a dry-run message locally if no webhook is configured).

7. **CI wiring** (`.github/workflows/eval.yml`) — runs the whole pipeline automatically
   on every pull request that touches the prompt, dataset, or classifier code, and
   exits with a failing status on critical regressions — blocking the merge.

## The hardest bug I hit (and the fix that mattered)

The first version compared each run against "whatever ran last," stored as local
timestamped files. That worked fine on my laptop — but in GitHub Actions, every run
starts on a brand-new machine with no memory of past runs. CI kept reporting "first
run, nothing to compare" even when a real regression was sitting right there,
completely silent. The fix was to commit one `baseline.json` file to git, so CI and
my laptop are always comparing against the exact same fixed point — and to make
updating that baseline a deliberate, separate step (`update_baseline.py`), never
something that happens automatically after a run.

That's the kind of bug that only shows up once you actually run something in CI
instead of just locally — which is why I made sure to actually exercise the real
GitHub Actions pipeline, not just trust that the logic "should" work.

## The result

I demoed the full lifecycle on a real pull request: broke the classifier on purpose,
watched the CI check fail with a clear regression report, fixed it, and watched the
same PR's check turn green and get merged — proving the gate actually catches
regressions and actually unblocks once they're fixed.

## What I'd highlight if asked "what was the hardest design decision?"

Deciding what counts as ground truth. It would have been easy to use an LLM to
generate the golden dataset's expected answers — but then the eval system would just
be checking if one model agrees with another, not whether the feature is actually
correct. Writing all 10 test cases by hand, including intentionally hard edge cases
like sarcasm and non-English input, was slower but is what makes the pass/fail signal
trustworthy.

## What I'd build next

- LLM-as-judge scoring for summary quality, replacing the current word-overlap
  heuristic with something more semantically aware
- Drift detection across a rolling window of runs, not just single-run vs. baseline
- Statistical significance testing so small, noisy deltas don't trigger false alarms

## Companion project

This project pairs naturally with [LLM Cost Autopilot](
https://github.com/Anas-Amiar/llm-cost-autopilot) — that one optimizes *cost* on every
request; this one catches *quality* regressions when a prompt or model changes.
Together they represent the two main levers an AI engineering team tunes in
production: correctness and cost.
