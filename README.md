# Can an LLM predict what a change will break? An answer key from GitHub history

Every "AI reviews your tickets" demo has the same hole: the output reads well
whether or not it is true. Plausibility is the one thing a language model is
reliably good at, so grading on it measures nothing.

This repository holds the evaluation material behind
[Premortem](https://marketplace.atlassian.com/apps/3455340266/premortem-risk-assessment-for-jira),
a Jira app that assesses an issue before the work starts. It is published so
the numbers below can be checked rather than believed — including the two
experiments that went against the product.

The model throughout is `gpt-5.5`. That matters: every miss recorded here was
made by a frontier model, so "use a better model" is not an available excuse.

---

## 1. The method: borrow the answer key from history

Open-source trackers write the answer down for you. Somewhere in every large
repository there is an issue that says *"regression introduced in #39177"*.
That sentence is a free, retrospective answer key: it names what broke, and it
names the change that caused it.

1. Search large repositories for issues whose body says *introduced in #N*,
   *caused by #N*, *regression from #N*, *broken by #N* or *after PR #N*,
   filed 2023-01-01 to 2025-08-31.
2. Take the **change description** — the author's own words, boilerplate
   stripped — as the ticket.
3. Assess it blind, with no sight of the follow-up issue.
4. Ask one question that has a real answer: **did the assessment name the
   failure the follow-up issue describes?**

423 candidates, 17 with an extractable reference, 6 kept for range. CI-flake
pairs were dropped: a flaky test is not a risk of the plan.

### The six pairs

Links only — the ticket text belongs to its authors.

| # | The change | What broke afterwards | Graded |
|---|---|---|---|
| 1 | [airflow#39177](https://github.com/apache/airflow/pull/39177) — serve logs when no remote logs are available | [#41164](https://github.com/apache/airflow/issues/41164) — server logs stopped being fetched when remote logs exist | near |
| 2 | [airflow#37736](https://github.com/apache/airflow/pull/37736) — dag/task ID as BigQuery job labels | [#39567](https://github.com/apache/airflow/issues/39567) — fails for task IDs of 64 characters | **hit** |
| 3 | [airflow#31833](https://github.com/apache/airflow/pull/31833) — check for a trailing slash in `base_url` | [#32996](https://github.com/apache/airflow/issues/32996) — `log_url` overwrites the existing path | **hit** |
| 4 | [elasticsearch#103581](https://github.com/elastic/elasticsearch/pull/103581) — "tidy up ImmutableOpenMap and Iterators a bit" | [#108049](https://github.com/elastic/elasticsearch/issues/108049) — deleting files in S3 trips `AccessControlException` | miss |
| 5 | [elasticsearch#97557](https://github.com/elastic/elasticsearch/pull/97557) — run downsampling as persistent tasks | [#98569](https://github.com/elastic/elasticsearch/issues/98569) — downsampling hits a security exception | hit, but see below |
| 6 | [prometheus#10916](https://github.com/prometheus/prometheus/pull/10916) — keep relabeled scrape interval on reloads | [#13910](https://github.com/prometheus/prometheus/issues/13910) — a reload metric gets inconsistent labels | near |

### The score

**Two hits, two near-misses, two misses out of six**, blind.

A later round scored three, but case 5 had stopped being blind: a prompt line
had been written after watching it fail. It demonstrates that the fix worked;
it is not independent evidence, and it is not counted.

**A hit, case 2.** The ticket said "set dag ID and task ID as BigQuery job
labels". The assessment asked what happens when an ID exceeds the label length
the API accepts. Two months later: *operator fails for task IDs of 64
characters*.

**A miss, case 4.** "Tidy up ImmutableOpenMap and Iterators a bit" produced
sensible, generic refactoring risks. The real failure — an access-control
exception deleting S3 files, because an iterator change moved code into a
different security context — was not among them, and nothing in the ticket
hinted at it. A human reviewer without knowledge of that codebase would have
missed it too. The ceiling here is the information in the ticket, not the
model.

---

## 2. The experiment that went against the product

Each risk carries severity, occurrence and detection ratings, and a priority
computed from them with the automotive Action Priority lookup — chosen because
multiplying ordinal ratings (FMEA's RPN) is mathematically meaningless.

The obvious next step is to sort the list by that priority. So it was
measured, on five held-out tickets whose real failure was known, by **where
the risk that actually came true ended up in its list**:

| ordering | rank of the real failure, per ticket | mean |
|---|---|---|
| the model's own order | 1, 1, 2, 1, 1 | **1.2** |
| likelihood, then impact | 2, 4, 1, 2, 3 | 2.4 |
| severity, then the model's order | 3, 5, 3, 1, 2 | 2.8 |
| computed Action Priority | 3, 7, 3, 1, 2 | 3.2 |

Every sort made it worse. The model writes the risk that matters first, and
each reordering threw that signal away.

Five tickets is not proof, and it is the first thing to re-measure on fresh
ones. But the data says: keep the ratings, show them, do not sort by them. The
product does exactly that — the feature its author was most pleased with is
the one the measurement said to switch off.

### Why the ratings changed shape

Asked for likelihood and impact on a low/medium/high scale, the model answered
**"medium" for 78% of 96 risks**, including all five that came true. A list
sorted on that is sorted on nothing.

Replacing it with three anchored 1–5 ratings — what has to be true for this to
happen, the worst realistic consequence, how late anyone would notice —
produced numbers that at least vary. Whether they are *calibrated* is a
different question, and only outcomes recorded over time can answer it.

---

## 3. Silence is a feature, and it has a number

The failure mode nobody demos is the one that kills these tools in practice:
risks on changes that are fine. A tool that writes eight paragraphs about a
typo fix teaches its users to skim, and within a week they skim the real ones
too.

So the suite has negative controls — changes that did not break anything. The
target is not "few risks", it is **nothing at all**.

| fixture | what it is | result |
|---|---|---|
| [`negative/BT-43.json`](negative/BT-43.json) | renaming a tab label, scope stated | **0 risks, 27 output tokens** |
| [`negative/BT-44.json`](negative/BT-44.json) | a typo and a hard-coded currency symbol in copy | **0 risks, 27 output tokens** |
| [prometheus#13937](https://github.com/prometheus/prometheus/pull/13937) | a real, additive feature in the same run — the control for over-correction | 8 risks, 1 605 output tokens |

The third one is the important one: silence on trivia must not become silence
on substance. It stayed at 8 risks before and after the rule that produced the
zeros.

That rule cost one paragraph in the prompt and removed an entire category of
noise. It is the only change in six rounds that told the model to produce
*less*, and it bought more than the three before it, all of which told it to
look harder.

---

## What is deliberately not here

- **The prompt and the harness.** They are the product. The fixtures and the
  answer keys are what make the claims checkable; the prompt is what makes
  them reproducible by a competitor.
- **The code-review benchmark data.** The code-risk side is also evaluated
  against [AACR-Bench](https://github.com/kagisearch/aacr-bench) pull requests
  with defects known in advance. That dataset belongs to its authors — use it
  from the source rather than a copy.
- **Ticket text from other people's projects.** Linked, not copied.

## If you are building something similar

- **Find an answer key before you build the grader.** Trackers, incident
  reports, post-mortems and reverted commits are retrospective ground truth,
  and they are free.
- **Grade on a question with a real answer.** "Did it name what broke" beats
  any rubric about helpfulness.
- **Measure your sorting.** Ordering is a model you did not know you were
  building, and it can be worse than no ordering at all.
- **Put benign inputs in the suite.** If silence is not measured, it does not
  survive the next prompt change.
- **Publish the misses.** Two of six is a number that can improve. "It writes
  great risks" is not.

---

The app: [Premortem — Risk Assessment for Jira](https://marketplace.atlassian.com/apps/3455340266/premortem-risk-assessment-for-jira),
free, runs on your own AI key.
Documentation: https://usepremortem.com/
