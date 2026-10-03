# Evolving the Interaction Policy of a Personalized Agent

MAS.S62 *Self-Evolving AI* (Fall 2026) course project.
**Zining Liu** (ziningl@mit.edu) · **Yangzi Sun** (sunyz@mit.edu) — Massachusetts Institute of Technology

## What this is

Two lines of work have improved personalized assistants. One makes the assistant
*proactive*: it acts on hidden intents and asks a clarification question when a request is
underspecified. The other makes it *self-evolving*: it improves its own components from
experience. Both leave out two features of real use. At the first interaction the agent has
no memory of the user (**cold start**), and over time the user's preferences change
(**preference drift**). In both situations what the agent does is decided by its
**interaction policy**: which preference to ask about, when to ask, and what to write to
memory. In every system we surveyed this policy is fixed, and it asks only when memory holds
nothing relevant, so the agent stops asking once its memory is confident and outdated.

We move self-evolution from the memory to the interaction policy. The policy is a short
**rule list** that maps the state of memory to an action. In prior systems these rules are
fixed; here an LLM proposes and rewrites them and the harness scores them, at two scales that
form one pipeline:

| | | runs it |
|---|---|---|
| **Idea 1: evolved across users** | one rule list evolved across *previous* users before deployment and run unchanged on a new user through all four phases | Liu |
| **Idea 2: revised within a user** | the rule list of Idea 1 kept changing while serving one user; whenever the agent acts without asking and post-action feedback arrives, a rule is added, removed, or a number in one is changed (for example, "ask again once a note is 30 tasks old" becomes 15 for a user whose preferences change often) | Sun |

A policy is judged by its **asking curve**: the questions it asks over the user's lifecycle
set against the questions that were needed. The area between the two curves, the
**asking-curve gap**, split into over-asking and under-asking, is the headline score.

## AskCurve, the evaluation protocol

AskCurve is built on PAHF's benchmark (arXiv 2602.16173). We keep PAHF's simulated users,
tasks, and harness, and add what the interaction policy needs. The full specification is in
[BENCHMARK.md](BENCHMARK.md).

- A user is a list of preference slots, each with a stake (reversible or irreversible) and a
  default; users differ by at least threefold in how often their preferences change.
- Each user is served 60 tasks; memory passes through cold start (1–10), warm (11–25),
  steady (26–40), and drift and recovery (41–60) after a scheduled drift at task 40.
- At each task the agent chooses `ask` (one user turn), `confirm` (half a turn), or `act`
  (no turn, then post-action feedback that is wrong with a set probability).
- The harness labels a question *needed* from the hidden preferences; tasks where the
  context already resolves the slot, so asking is wasted, are included on purpose.
- Six baselines on every split: never ask, always ask, PAHF's fixed rule, asking again at a
  fixed interval, a one-step value-of-information policy, and the evolved policy.
- One check stated in advance: in the drift phase PAHF's fixed rule must ask fewer than one
  in ten of the needed questions, or the thesis is wrong and we report that.

**Data.** Version 0 instantiates the protocol on PAHF's two domains, a home assistant and an
online shopping assistant (40 and 20 users, re-split 3:1 into previous users, used for
evolution, and test users). Version 1 adds a second domain in the style of CAPA
(arXiv 2607.26611), a coding assistant with recurring personal ambiguities (60 users, 20 for
testing).

**Metrics.** Asking-curve gap (headline); silent-error cost; gain per early question;
detection lag and questions to recovery; repeat-question rate and spurious-write rate.

**Risks.** (1) PAHF's simulated users come from one template, so an evolved policy may be a
fixed schedule; test users and the second domain check this. (2) At cold start all policies
may look alike (Pep bounds the gain of question choice at about one question per
interaction); the expected difference comes after drift, so the gap is reported per phase.

## Layout

```
harness/      PAHF fork plus the seven protocol additions (needed labels, stakes and defaults,
              context-resolvable slots, confirm, drift schedule, feedback noise, one log format)
policy/       rule list schema, the evolution loop across users (Idea 1), the revision loop
              within a user (Idea 2)
scorer/       asking curve and the seven metrics from the harness log
baselines/    never ask · always ask · PAHF's fixed rule · fixed interval · one-step VOI
experiments/  run configs and logs, one directory per dated run
```

## Team and timeline

Liu designs the protocol and the evolution loop and runs the experiments of Idea 1; Sun
reproduces PAHF, extends its harness, builds the second domain, and runs the baselines and
the experiments of Idea 2; analysis and the reports are joint.

| dates | milestone |
|---|---|
| Oct 6–13 | fork PAHF, reproduce its four feedback settings, add the seven protocol elements |
| Oct 14–21 | the five fixed baselines on PAHF and a first run of Idea 1's loop; midterm presentation Oct 21 |
| Oct 22–Nov 4 | Idea 1: the loop on previous users, scored on test users |
| Nov 5–18 | Idea 1 on the second domain; Idea 2's loop within each user |
| Nov 19–Dec 2 | Idea 2 on the second domain; final presentation Dec 2 |
| Dec 3–9 | final report |

## Status

Proposal stage (2026-10-02). The protocol specification is complete; code lands from Oct 6.
