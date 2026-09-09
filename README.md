# LLM-Assisted QC for Quantitative Image Analysis

A human-in-the-loop pattern that uses a large language model to triage and explain
quality-control issues in a measurement pipeline — while structurally preventing the
model from touching the measurements or any decision that affects the result.

I built this to solve a real problem in my own quantitative-analysis work, and I use it
day to day. It is a working single-user tool rather than a productionized team system,
and this writeup is honest about that. It describes the design, the reasoning behind it,
and how I evaluate it. Where outcomes are not yet formally measured, I say so.

Study-specific details have been deliberately generalized; this is shared as a worked
example of quality-control methodology, not as a description of any particular dataset
or experiment.

---

## The problem

The pipeline quantifies signal across several measurement channels from imaging data,
producing numeric endpoints that are ultimately compared across multiple experimental
groups. Before those measurements can be analyzed, someone has to:

- review the exported measurements,
- reconcile metadata against an approved reference sheet,
- inspect suspicious results against the source images, and
- decide which measurements are genuinely suitable for analysis.

The hard part is the reason this can't be fully automated: **an extreme value can
reflect either a real signal or a technical failure, and the two demand opposite
responses.** A zero can mean a genuinely low true value, an empty region, or a failed
detection step. If exclusions get applied inconsistently across groups, the comparison
between conditions is silently biased. The cost of a wrong call isn't a slow afternoon —
it's a corrupted result that still looks clean.

One methodological point that shapes what the model is allowed to conclude: different
endpoints carry different valid interpretations. A raw "positive area" is not the same
claim as a validated object count, and the system has to keep those distinct rather than
letting the model collapse one into the other.

## What the system does — and what it deliberately does not

Deterministic, rule-based code does the numeric work. The LLM (currently via the OpenAI
API) sits on top of the structured outputs and handles the *interpretation and
communication* layer only.

**The LLM's job:**

- Explain each QC flag in plain language.
- Group related problems — e.g. a single missing calibration value affecting an entire
  batch.
- Suggest metadata corrections **supported by the authoritative reference sheet**.
- Prioritize which records most need human review.
- Summarize accepted data, exclusions, and unresolved issues.
- Explain statistics produced by validated analysis code.

**What the LLM is explicitly forbidden from doing:**

- Inventing missing calibration or metadata values.
- Inferring group assignment from the appearance of the data.
- Adjusting thresholds so that groups agree.
- Deciding that an unusual but valid measurement "must be" an error.

This boundary is the whole design: **the model never touches a measurement or a decision
that affects the comparison. It triages, explains, and routes to a human.**

## Pipeline

```
Raw exports
   → rule-based QC (numeric checks, deterministic)
   → LLM review summary (explanation, grouping, prioritization)
   → human decisions (accept / correct / reprocess / exclude)
   → approved dataset
   → scripted analysis (validated code)
   → evidence-linked narrative
```

## Prompt and output design

The model runs against a structured prompt with a strict, auditable output format. The
instruction is deliberately restrictive:

> Review these measurements using only the supplied QC rules, metadata, and analysis
> settings. Distinguish technical failures from unusual but potentially valid
> measurements. Do not infer missing metadata, replace missing values with zero, remove
> outliers, or change measurements. Every recommendation must reference a record and
> supporting evidence. When evidence is insufficient, request review.

Every issue returned is a structured record with fixed fields:

`record_id` · `qc_code` · `severity` · `evidence` · `affected_endpoint` ·
`suggested_action` · `review_status`

Requiring per-record evidence and an explicit "request review" path is what stops the
model from confidently papering over uncertainty — if it can't cite a reason, it must
escalate rather than guess.

### Representative QC rules the model reasons over

| Condition | Handling |
|---|---|
| Missing required columns | Stop the affected import; list missing fields |
| Missing key identifier or group metadata | Hold records from group comparison until verified |
| Missing area or calibration/scale | Block scale-dependent endpoints; preserve other valid outputs |
| Duplicate record rows | Flag possible duplication; compare source + run/version before resolving |
| Suspicious zero values | Request source-image review; **do not** auto-exclude |
| Out-of-range region size | Flag against a predefined, context-specific rule |
| Empty region / failed detection | Mark measurement invalid — not a true zero |
| Missing baseline reference | Block only baseline-dependent results; keep valid absolute measurements |
| Locked output file | Report the write failure; offer a safe retry/export |
| Oversized input | Use validated tiled processing or fail explicitly; never silently change resolution |

Group-count reporting shows **both** total measurement counts and *distinct-subject
counts* across each grouping factor, so repeated measurements from one subject can never
be mistaken for independent samples.

## Failure modes I designed against

These are the specific ways an LLM can quietly corrupt this kind of analysis. I treat
each as something the system must structurally prevent, not something I hope the model
gets right:

- **Mistaking a failed detection step for a real low value.** A result reads zero
  because a processing step failed; a naive model interprets it as a genuinely low
  measurement. An explicit processing-status field blocks that reading.
- **"Cleaning away" a real effect.** A value is far larger than the rest; the model
  recommends excluding it *because* it's extreme. The correct action is technical
  review, with retention if the measurement is valid.
- **Inventing normalization.** A baseline reference is missing, so the model substitutes
  another group's mean. Baseline-dependent output must simply stay unavailable.
- **Inflating sample size.** Many measurements from a single subject get described as
  many independent replicates. The subject → sample → region → measurement hierarchy has
  to stay intact.

I also probe contradictory metadata, inconsistent units, source/channel swaps, and
accidental inclusion of multiple analysis runs for the same record.

## The human-review loop

The review screen shows, together: the source data crop, the processing overlay, the
measurement, the QC reason, and the suggested action. The reviewer chooses
**accept / correct metadata / reprocess / exclude**, with a required reason for any
change or exclusion.

Review discipline I hold myself to:

- Review every proposed correction and exclusion.
- Inspect all flagged records **plus a random sample of unflagged ones**, so the QC
  layer's own blind spots surface.
- Keep the group label hidden during technical review where practical, so the technical
  call stays blind to the hypothesis.
- Establish processing settings on a representative reviewed set, then **lock the
  procedure before any group comparison.**
- Preserve originals, settings, software versions, and every decision.

A structural point the model cannot resolve on its own: if one group was processed
entirely in a single batch, group is confounded with batch. The system flags that
confounding; it does not pretend the numbers can fix it.

## Outcome and how I evaluate it

**Status: built and in personal use. Outcomes are not yet formally measured.** I'm
stating that plainly rather than quoting a number I haven't earned. The intended outcome
is more consistent, auditable measurement with less repetitive review, while preserving
genuine variation in the data.

The metrics I evaluate it on:

- Review time per batch, including rework.
- Technical problems the rule-based QC missed.
- Incorrect correction or exclusion recommendations from the model.
- Agreement with independently reviewed processing and QC decisions.
- Usable subject counts in each experimental group.
- Stability of conclusions under predefined threshold and exclusion sensitivity checks.

The statistical plan accounts for repeated within-subject measurements and batch
structure; the appropriate model depends on the endpoint, the sample size, and how
treatment was allocated across subjects.

## Why I built it this way

The design reflects one conviction: **for data that feeds a decision — a scientific
conclusion, or the training of a model — the quality layer is where correctness is won
or lost, and an LLM is powerful there precisely because it is kept away from the
numbers.** It explains, triages, and routes; humans decide; validated code computes.
That separation is what makes the output trustworthy enough to build on.

---

*Built and maintained by Jake Vaynshteyn. Single-user tool; shared as a worked example
of human-in-the-loop LLM quality control. Study-specific details generalized by design.*
