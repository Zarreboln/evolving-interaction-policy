# Evolving the Interaction Policy of a Personalized Agent

MAS.S62 *Self-Evolving AI* (Fall 2026) course project.
**Zining Liu** (ziningl@mit.edu) · **Yang Zi Sun** — Massachusetts Institute of Technology

## What this is

Personalized agents have scaled what they *remember* — longer contexts, structured memory
banks, retrieval over months of history. In genuinely interactive deployments that scaling
stops paying, because the limit is no longer what the agent can recall but how it
*interacts*: when it spends a user turn on a question, and which corrections it admits into
memory. Across the systems we surveyed, that decision is hand-written and conditioned on the
memory being **empty** rather than **wrong**, so the agent stops asking exactly when its
memory has filled and gone stale.

Self-evolution has reached the memory. It has yet to reach the interaction with the user.

We place the interaction policy itself inside the evolution loop, at two scales:

| | | lead |
|---|---|---|
| **Idea 1** | Cold-start initialisation — the policy is evolved across *past* users and deployed unchanged from the first turn | Sun |
| **Idea 2** | Adaptation under drift — the same rules are edited *inside* one user from that user's own corrections | Liu |

The evolved artifact is one small set of written rules in a `When [condition], [action]`
form, deciding whether the next turn is spent on a question and which attribute it should
concern. Conditions are computed in the harness from observable state: whether an entry
exists for an attribute, its age, whether a correction touched it recently, and the user's
estimated drift rate. Rules are proposed, scored by the accuracy they buy *per question
asked*, and kept or retired.

## Benchmark and metrics

Evaluation runs on **PAHF** (arXiv 2602.16173), the only released benchmark with both a
pre-action clarification channel and a post-action correction channel under controlled
preference drift, plus **VitaBench 2.0** for the per-user axis (drift rates differ ~3.3x
across users).

PAHF's feedback-frequency metric merges asking with correction, which is the distinction our
claim rests on, so we add: gain per early question, `FF_pre` / `FF_post` counted separately,
questions to recover (right-censored at 12), and a spurious-write rate under injected false
corrections.

Expected conclusions are pre-registered in the proposal, including the negative result: if
the gain is flat across strategies, the signal sits in the world model rather than in the
policy, and we report that rather than re-tuning the comparison.

## Layout

```
harness/      forked PAHF harness, backend model swap, single OpenAI call site
policy/       rule schema, proposal and retirement loop, per-user drift estimate
scorer/       gain per question, FF_pre / FF_post, questions to recover, spurious writes
baselines/    no user model · fixed onboarding · ask-when-empty (PAHF) · evolved policy
experiments/  run configs and logs, one directory per dated run
```

## Status

Proposal stage (2026-10-02). Code lands from the week of 6 Oct: fork the harness, swap the
backend model, reproduce all four PAHF baselines on the embodied domain, and freeze the rule
schema and the scorer before either evolution loop is written.
