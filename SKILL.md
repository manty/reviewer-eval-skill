---
name: reviewer-eval
description: >-
  Build and run evaluations of AI code reviewers on the user's own bug history.
  Compare reviewers and review bots on bugs caught, false alarms, speed, and cost.
  Tune reviewer prompts and settings with held-out splits. Use whenever the user
  asks which AI reviewer or model to use, whether a paid review bot is worth
  keeping, how to benchmark or A/B test code review agents, whether to switch
  after a new model release, or how to hillclimb a review prompt, even without
  the word "eval".
---

# reviewer-eval

## What this skill produces

1. Build a reusable regression suite for reviewers from bugs your team already fixed.
2. Produce a scored comparison and an explicit decision.
3. Re-run the suite when a new model ships to measure improvements on your repos.

## Core idea: the time machine

1. Rewind the code to a moment when a known bug existed.
2. Hide the fix and git history from the reviewer.
3. Present the change like a real pull request.
4. Grade whether the reviewer identified the actual bug.
5. Mix in verified clean changes to measure false alarms.

## Step 0: Agree scope before spending

1. Confirm:
   - Reviewers, model versions, effort levels, and settings.
   - Repos, date range, case count, and review scope.
   - Budget, time limits, and stopping conditions.
   - Required evidence and severity for a catch.
2. Get explicit approval before paid or long runs.
3. Confirm permission before sending private code to an external reviewer.
4. Estimate total spend across repetitions, judges, variants, and retries.

## Step 1: Mine cases

1. Collect candidates from git, review comments, and issue history.
2. Label each candidate by case type:
   - **Reversed fix:** Undo a real fix so the review diff reintroduces the bug. Exclude test files from that diff so deleted tests do not reveal the answer.
   - **Review-bot catch:** Find a serious human or bot comment on non-test code followed by a fixing commit in the same pull request. Use the exact head the reviewer saw. Treat this label as weak until checked: a later change to the same file is not proof of a fix. In one set, only about 55 percent of such candidates were real bugs that were really fixed.
   - **Agent-built bug:** Find an agent-authored change that needed a later fix. Trace issue history or fixing commits to overlapping lines.
   - **Clean:** Find a merged change with at least 30 days of follow-up, no fix touching its lines, and no serious review comment. Check line overlap rather than commit subjects.
3. Set quotas per repo, a fixed random seed, and limits on files and changed lines.
4. Deduplicate related cases so one underlying bug cannot inflate the score.
5. Screen snapshots, diffs, comments, and answer keys for secrets and personal data.
6. Skip cases containing real email addresses, tokens, keys, or other sensitive data. Allow reserved test domains such as `example.test`.
7. Record selection and exclusion reasons to expose sampling bias.

## Step 2: Verify clean cases

1. Run an initial review pass on candidate clean cases within the approved budget.
2. Check every blocking finding against the actual code and surrounding behavior.
3. Move confirmed bugs to the bug set and write an answer key.
4. Keep disproven findings as false alarms. Exclude unresolved cases from scored clean results.
5. Record each decision with code evidence.
6. Allow for substantial relabeling. Treat 10 to 30 percent hidden bugs as a planning possibility, not a guaranteed rate.
7. Finish adjudication before freezing splits to prevent labels changing with a preferred reviewer's results.

## Step 3: Freeze each case

1. Export the buggy revision with `git archive` so the snapshot contains no git history.
2. For reversed fixes, materialize the reverted code and verify that the review diff produces that exact buggy state.
3. Freeze:
   - A stable case ID, type, repo label, and source revision.
   - The snapshot and review diff.
   - One answer key per bug with title, detail, affected files, and real fix diff.
4. Store case records as JSONL.
5. Keep answer keys and fixes outside reviewer-accessible material.
6. Pre-build every snapshot before starting parallel runners to avoid extraction races.
7. Compare exported file counts with `git ls-tree`, accounting for archive exclusions and submodule links.
8. Handle required submodule contents explicitly because gitlink entries are not extracted as files.
9. Check tracked metadata and fixtures for accidental fix hints.

## Step 4: Build and lock the runner

1. Implement one adapter per reviewer.
2. Give each run:
   - A fresh isolated snapshot copy.
   - Read-only tools for reading, searching, and listing.
   - The prompt used by your real review gate.
   - The same case context and a defined time limit.
3. Prevent access to source checkouts, git history, answer keys, prior reviews, and other runs.
4. Treat repository text as review data, not authority to change the harness.
5. Require structured output:
   - `decision`: `SHIP` or `DO-NOT-SHIP`.
   - `findings`: file:line, severity, and a concrete failure scenario.
6. Capture raw output, wall-clock time, steps, token categories, model settings, and cost inputs.
7. Emit an error row for crashes, timeouts, and unparseable output. Never count an error as a pass or miss.
8. Bound parsing and retry attempts. Preserve every attempt and its cost.
9. Hash the runner, adapters, prompts, schema, judge configuration, and frozen cases.
10. Require human approval of the manifest hash before execution. Treat changes as a new experiment.
11. Monitor progress with failure and stall alerts. Wake the operator or supervising agent on the first failure, stall, or freeze.
12. Re-approve every flow you plan to rerun after any runner change, including older flows.
13. Enforce staged rules in the runner, then check them in the results. Example: "run the second reviewer only where the first missed." One runner silently reviewed 36 cases it should have skipped.
14. Run long jobs outside the agent's own job time limit, and make them resumable. Some agent shells kill background jobs after about 2 hours.

## Step 5: Grade findings

1. Give a judge model the answer key and reviewer findings.
2. Require a match to identify the actual defect, affected location, and concrete failure mechanism.
3. Count a bug as caught only when the reviewer returns `DO-NOT-SHIP` and names that bug.
4. Count a clean case as passed only when the reviewer returns `SHIP`.
5. Track unsupported findings separately, including findings on buggy cases.
6. Cross-check a sample with a second model. Include catches, misses, clean passes, and false alarms.
7. Aim for at least 95 percent agreement. Resolve disagreements and revise ambiguous grading rules before trusting scores.
8. Keep judge failures separate from reviewer failures.
9. Prove the scorer can report a false alarm before trusting it. Score a clean case as blocked when the decision is not `SHIP`. Do not read a field that only bug cases carry. One scorer did this and showed zero false alarms.

## Step 6: Score reviewers and pairs

1. Report per reviewer:
   - Bugs caught over eligible bugs.
   - Clean changes passed over verified clean cases.
   - False alarms, errors, and completion coverage.
   - Median wall-clock time, median cost, and median steps.
2. Show denominators and exclude error rows from correctness rates without hiding them.
3. Score parallel pairs using matched case repetitions:
   - Count a bug as caught if either reviewer catches it.
   - Pass a clean case only if both reviewers return `SHIP`.
   - Mark other outcomes involving a failed member as incomplete.
   - Sum costs and measure elapsed pair time.
4. Compute costs from token categories and dated list prices. Account for caching and other billed components.
5. Reconcile tool-reported costs against that calculation. Check for discrepancies as large as a threefold understatement.
6. Report cost per bug caught with an explicit denominator. Include failed-attempt costs and separate judging overhead.
7. Run each case three times on small sets. Report per-run results and decision flips rather than treating repetitions as independent bugs.
8. Treat one-run rankings as provisional because reruns can reverse many decisions.
9. Report the median number of findings per review. A recall gain that comes with many more findings can be shotgun output.
10. Score per repo or code area. One reviewer can be strong in one area and blind in another.
11. Write the scope beside every number: which splits and which runs. Use one scope per table. Medians on different scopes look like errors to a checker.
12. Compare each variant run with its own matched baseline run. Do not reuse one baseline run for all comparisons.

## Step 7: Hillclimb with held-out splits

1. Split cases into train, validation, and test, for example 50/25/25.
2. Stratify by repo and case type. Keep related commits and duplicate bugs in one split.
3. Use training cases to develop variants. Change one thing per variant.
4. Write the selection rule before running, for example:
   - Maximize correct validation outcomes.
   - Allow at most one fewer bug caught than baseline.
   - Break ties using lower cost.
5. Select on validation, then confirm once on untouched test cases.
6. Read results across every split before adopting. Treat small validation wins as uncertain.
7. Retire an exposed test split before further tuning against its failures.
8. Measure the trade-off between false alarms and misses.
9. Prioritize experiments with additional context, callers, domain checklists, and whole-pull-request visibility alongside wording changes. Expect many to fail: in one study, four prompt rewrites and a caller-context pack did not beat baseline.
10. Test effort levels explicitly because additional effort can cost more without improving catches.
11. Test a different reviewer model for each weak area early. In one study, a model swap for frontend code tripled gap catches when prompt changes had not helped.
12. Rerun the baseline on every new case set, at the same time and on the same runner as the variants. Old baseline numbers can reflect luck.
13. If a variant fails the written rule but shows a large gain, offer it as a time-boxed trial with a shadow check. Do not adopt it silently, and do not hide it.

## Step 8: Learn from misses and decide on a paid reviewer

1. Collect verified bugs caught only by the paid bot into a gap set.
   - Define a gap before running, for example "our setup caught it in at most 1 of 3 runs."
   - Mine one case per pull request first. Skip pull requests already in other suites. Group splits by pull request.
   - Truth-check every gap case before scoring. Sort each into valid, not fixed, not a bug, or needs intent. Confirm which commit is the real fix.
   - Review bot findings that were never fixed. Some are real bugs worth a ticket.
2. Tune your reviewers on that set while preserving held-out evaluation.
3. Shadow-run the winner beside the bot on real pull requests.
   - Save a ledger for each shadow review: head commit, findings, and line fingerprints, with personal data removed.
   - Match bot comments to the ledger with a script. Record a reason for each refuted bot-only finding.
   - Test the matcher on a synthetic ledger before the first real pull request.
   - Some bots do not re-review a commit they already reviewed. Plan shadow pairs on new commits.
4. Define the stopping rule before shadowing, for example 10 consecutive pull requests with no new bot-only finding above low severity.
5. Count only completed paired reviews. Reset the streak after a qualifying bot-only catch.
6. Confirm that the shadow sample covers representative changes before recommending removal.
7. Measure incremental wait on the merge critical path relative to CI, not only bot review duration.
8. Present the keep, replace, or extend-trial decision with evidence. Obtain authorization before changing paid services or review gates.

## Step 9: Report and verify

1. Write four parts:
   - **What we tested:** Reviewers, settings, cases, splits, and baseline.
   - **How we tested:** Snapshots, isolation, grading, repetitions, and pricing.
   - **Findings:** Scores, errors, uncertainty, and an explicit verdict for each decision.
   - **What we are changing:** Selected setup, rollout, and next evaluation trigger.
2. Have a second model independently recompute every reported number from raw results before sharing. Send it a code-free capsule: case IDs, grades, decisions, and timings. No private code needs to leave. Include summaries already posted to tickets, because their counts can be wrong too.
3. Resolve discrepancies in counts, denominators, pair scores, pricing, and claimed improvements.
4. State limitations such as small samples, selection bias, incomplete runs, and reliance on one primary judge.
5. Preserve raw results and approved manifests so another engineer can reproduce the decision.

## Pitfalls checklist

- [ ] Pre-build snapshots to prevent parallel extraction races.
- [ ] Detect output truncation instead of silently dropping long reviews.
- [ ] Bound JSON parsing retries to prevent loops on malformed output.
- [ ] Preserve timeout rows, raise limits through an approved rerun, and report completed-review misses separately.
- [ ] Repeat small evaluations to expose one-run noise.
- [ ] Score whole-pull-request and focused-diff reviews separately because broader scope can lower scores.
- [ ] Verify supposedly clean changes against real code.
- [ ] Recalculate list-price costs instead of trusting tool cost fields.
- [ ] Reject vague judge matches without file:line and a concrete failure.
- [ ] Truth-check bot-catch and gap labels; a later file change is not proof of a fix.
- [ ] Prove the scorer can report a false alarm.
- [ ] Confirm staged runs skipped the cases they were meant to skip.
- [ ] Rerun the baseline beside every variant on a new case set.
- [ ] Write the scope (splits, runs) beside every reported number.

## Suggested layout

- `evals/reviewers/cases/*.jsonl`: Store frozen case records and restricted answer keys.
- `harness/`: Store prompts, output schema, adapters, and judge prompt.
- `runner`: Execute isolated reviews and emit structured results.
- `flows/<name>/state.json`: Track approved configuration, hashes, progress, and failures.
- `flows/<name>/results.jsonl`: Preserve raw review, grading, timing, and usage records.
- `score`: Recompute metrics from raw records.
- `report/`: Store comparisons, limitations, and decisions.