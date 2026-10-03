# AskCurve: a lifecycle protocol for the interaction policy of a personalized agent

Status: specification, 2026-10-02. Terms follow the proposal report. v0 is built on the PAHF
harness; v1 adds a second domain. Nothing here is implemented yet.

## 1. What it measures

One question, asked over a whole user lifecycle: **at this turn, should the agent spend
the user's attention on a question, and on which preference?**

Existing benchmarks each hold one piece of this:

| benchmark | has | lacks |
|---|---|---|
| PAHF (2602.16173) | pre-action clarification + post-action feedback, one preference shift | needed-question labels, stakes, feedback noise, more than one drift, a confirm action; its feedback-frequency metric merges asking with correcting |
| CAPA (2607.26611) | fewer clarifications as session history fills | drift (the curve only goes down), stakes, any domain beyond coding |
| ATRBench (2605.28108) | asking now for a preference needed only later | drift, a current-task ask-vs-act decision |
| MCB (2608.19564) | remember / verify / ask as a three-way choice | a sequence; 140 static scenarios |
| HorizonBench, PERMA, CAPTURE | long-horizon drift, memory read/write | an asking action at all |

AskCurve puts them on one axis: the **asking curve** of the proposal, the per-task
record of questions *asked*, questions *needed*, and success, drawn over cold start,
warm, steady, and drift and recovery. Every metric below is a reading of that curve.

## 2. Formal setup

**User.** A user $u$ is a preference vector $\theta_u(t) \in \prod_{k=1}^{K} V_k$ over
$K$ attributes (slots), piecewise constant in task index $t$. Each slot $k$ carries:

- recurrence $r_k$: the fraction of tasks that depend on it;
- stake $s_k \in \{\text{low}, \text{high}\}$: cost of acting on a wrong value
  (reversible vs. irreversible outcome);
- a population default $d_k$, which is right for some users and wrong for others.

**Drift.** $\theta_u$ changes at change points. One **scheduled** drift at a fixed task
index aligns the phases across users; additional **unscheduled** drifts are sampled per
user at rate $\lambda_u$, drawn so that drift rates differ across users by at least 3x
(the per-user axis of Idea 2). A drift flips one or more slot values. Drift is visible to
the agent only through post-action feedback or by asking again.

**Task.** Task $x_t$ declares a dependency set $S_t \subseteq [K]$ and a context $c_t$
that reveals the correct value of a subset $R_t \subseteq S_t$ (context-resolvable
slots). Success requires the action to match $\theta_u(t)$ on every $k \in S_t$.

**Agent turn.** Observe $(x_t, c_t, M_t)$ where $M_t$ is the agent's memory. Choose one
of three actions:

| action | cost in user turns | returns |
|---|---|---|
| `ask(k)` | 1 | $\theta_{u,k}(t)$, truthfully |
| `confirm(k, v)` | 0.5 | yes / no |
| `act` | 0 | outcome; then post-action feedback |

Any number of `ask` and `confirm` actions per task; each costs its user turns, so the
asking curve counts questions per task, not tasks with a question.

**Post-action feedback.** With probability $p_{fb}$ the simulated user corrects one
wrong slot. With probability $\varepsilon$ the feedback is *incorrect* (injected noise).
The agent decides whether to write the feedback into memory.

**What is fixed.** The backbone LLM, the user simulator, the memory store and the task
sequence are frozen and seeded. The policy is the only moving part. An open track lets
the memory system vary too.

## 3. Ground-truth labels, computed by the harness

Labels are derived from the hidden $\theta_u(t)$, not annotated.

**Needed question.** Slot $k$ is *needed* at task $t$ iff

1. $k \in S_t$ (the task depends on it),
2. $k \notin R_t$ (the context does not reveal it),
3. memory has no entry for $k$, or its entry $\neq \theta_{u,k}(t)$,
4. $d_k \neq \theta_{u,k}(t)$ (acting on the default would fail).

In words: acting silently would fail on this slot and nothing observable repairs it.
Condition 4 is what stops "always ask when memory is empty" from being optimal; condition
3's second clause is what makes an outdated note count as missing.

**Four instance types** fall out of these conditions and are balanced by construction:

| type | memory | needed | tests |
|---|---|---|---|
| A complete | correct entry | no | over-asking in steady state |
| B missing, unrecoverable | none | yes | under-asking at cold start |
| C missing, recoverable | none, but $k \in R_t$ or $d_k$ is right | no | the hard negative: inference instead of asking |
| D outdated | entry $\neq \theta_{u,k}(t)$ after drift | yes | asking again after drift, the failure mode of PAHF's fixed rule |

**Worth-asking-early** (optional ATR track). $W_t(k) = $ number of tasks in
$(t, t+H]$ that depend on $k$. A question about a slot with high $W$ and no current need
is scored by its future gain, not penalized as over-asking.

## 4. Lifecycle phases

Per user, $T = 60$ tasks in a fixed order:

| phase | tasks | memory state | what the curve should show |
|---|---|---|---|
| P1 cold start | 1–10 | empty | asked $\approx$ needed, concentrated on high-$r_k$ high-$s_k$ slots |
| P2 warm | 11–25 | filling | asked falls faster than needed if inference works (type C) |
| P3 steady | 26–40 | full, correct | asked $\approx 0$; confirms only on high-stake slots |
| P4 drift and recovery | 41–60 | partly outdated after scheduled drift at $t=40$ | asked rises again, then falls; success recovers |

Unscheduled drifts land anywhere; they are what distinguishes users by $\lambda_u$.

## 5. Metrics

All are readings of the asking curve. Phase-resolved unless stated. Seven metrics; the
report (Section 4.3) lists the same seven.

**Headline.** *Asking-curve gap*: area between the asked and needed curves, split into
over-asking $\sum_t (\text{asked}_t - \text{needed}_t)^+$ and under-asking
$\sum_t (\text{needed}_t - \text{asked}_t)^+$. Report both; a single number hides which
way a policy fails. (Question precision / recall against needed questions are the same
two quantities normalized by questions asked / questions needed, so they are not listed
separately; the pre-registered check in Section 9 is the drift-phase recall.)

**Cost of silence.**
- Stake-weighted silent-error cost: $\sum_t \sum_{k \in S_t} s_k \cdot
  \mathbb{1}[\text{acted wrong on } k]$.

**Efficiency.**
- Gain per early question: P2 success-rate gain per P1 question (Idea 1 metric).

**Drift response.**
- Detection lag: tasks from a drift to the first question or confirm on a drifted slot.
- Questions to recovery: questions until pre-drift success rate is regained,
  right-censored at 12.

**Hygiene.**
- Repeat-question rate: same slot asked twice with no intervening drift or feedback.
- Spurious-write rate: incorrect post-action feedback written to memory.

**Logged but not scored.** PAHF's `FF_pre` / `FF_post` (pre-action questions and
post-action corrections) are kept in the per-task log for comparison with PAHF's own
numbers; they duplicate the asked curve and the unweighted silent-error count.
Calibration diagnostic: ask rate binned by the harness's own posterior uncertainty on
the slot; a calibrated policy asks more where it knows less.

## 6. Baselines

Seven baselines run on every split, in this order; the first four are controls, not contributions.

| policy | expected signature on the curve |
|---|---|
| Never ask | success = default-match rate; under-asking area maximal |
| Always ask (every slot the task depends on) | over-asking maximal; defines the ceiling of the asked curve |
| PAHF's fixed rule (ask about every slot memory lacks) | asked $\approx$ needed in P1, over-asks on context-resolvable slots in P2, **needed recall $\to 0$ in P4** |
| Ask again at a fixed interval | P4 recovers; P3 over-asks at the interval rate |
| One-step value of information (REVOIR-style, no memory of recurrence) | good per-task; under-invests in high-$W$ slots at P1 |
| Oracle (knows $\theta_u(t)$) | success $\approx 1$, zero questions; defines the floor of both areas |
| Evolved policy (ours) | the claim: smaller total gap than every fixed baseline except the oracle, over the whole lifecycle |

The PAHF's-fixed-rule row is the pre-registered demonstration of the proposal's thesis. If
it does not collapse in P4, our central hypothesis is wrong and we report that.

## 7. Domains and scale

| | v0 (Oct–Nov 2026) | v1 |
|---|---|---|
| domains | PAHF embodied (home assistant), PAHF online shopping | + a second domain (coding assistant, CAPA-style recurring ambiguity) |
| users per domain | PAHF's 40 / 20, re-split 3:1 previous (evolution) / test | 60, 20 test |
| slots $K$ | PAHF's attributes, with added $s_k$ and $d_k$ | 8–12 |
| tasks per user $T$ | 60 (PAHF's 4 phases re-cut) | 60 |
| drifts per user | 1 scheduled + Poisson($\lambda_u T$) unscheduled | same |
| feedback noise $\varepsilon$ | 0, 0.1, 0.3 | same |

Test users check transfer of an evolved policy (Idea 1); the second domain checks whether
the rules are domain-specific (Risk 1).

## 8. What v0 adds to the PAHF harness

Concretely, in `harness/`:

1. needed-question labels from the hidden preferences (Section 3);
2. per-slot stake $s_k$ and default $d_k$;
3. context-resolvable slots $R_t$ to populate type C;
4. a `confirm` action with half cost;
5. a drift schedule with per-user $\lambda_u$ on top of PAHF's single shift;
6. feedback noise $\varepsilon$;
7. `scorer/` emits the asking curve and every metric in Section 5 from one log format.

Everything else in PAHF stays as released, so PAHF's own numbers remain reproducible on
our fork as a check.

## 9. Validity checks, pre-registered

- Oracle reaches success $\geq 0.98$ with zero questions on every split; otherwise the
  labels or the simulator are wrong.
- Never-ask success equals the default-match rate within 2 points.
- PAHF's fixed rule's needed recall in P4 is below 0.1 on every split.
- Type-C instances make up 25–35% of tasks per phase; otherwise the hard negative is
  too rare to move precision.
- Re-running with a different seed changes no headline number by more than 1 point.

## 10. Open decisions

- Whether `confirm` belongs in the main track or only in an extension; PAHF has no such
  action, and adding it changes every baseline's numbers.
- The confirm cost (0.5 above) is a guess; a small pilot with a human-written cost
  survey would be better than a constant.
- Whether the ATR track is in scope this term. It is cheap once $W_t(k)$ is logged.
