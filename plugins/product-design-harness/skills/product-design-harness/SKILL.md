---
name: product-design-harness
description: Use when a product direction, feature, workflow, experiment, launch, feedback signal, automation, or scale decision needs judgment before execution or the next iteration.
license: MIT
---

# Product Design Harness

Decide whether a product direction should continue, be verified, or be
reframed anywhere in the product loop.

Execution includes design, implementation, prototype, launch, growth,
automation, or investment.

## Start

Load in this order:

1. `resources/schemas/session-config.schema.json`: set `working_language`; keep
   `canonical_identifiers` in English.
2. `resources/knowledge/ontology.json`: use the canonical UX3 knowledge kernel and its
   approved Flow definitions.
3. `resources/knowledge/rules.json`: load required rules plus only triggered conditional
   rules.
4. `resources/prompts/start-review.md`.

Then fill `resources/templates/product-brief.md`, select the smallest responsible mode,
and return `resources/schemas/review-result.schema.json`.

Validate every review result with `resources/scripts/check_review.py`. JSON Schema checks
shape and conditional fields; `check_review.py` is the canonical validator for
the worst-verdict, weakest-flow, and headline-tier semantic checks.

## Modes

| Mode | Use when | Source section |
|---|---| --- |
| Quick Gate | Small, reversible, low-risk change. | [Modes](#modes) |
| Standard Gate | New feature, workflow, or experiment. | [Modes](#modes) |
| UX3 Council | High uncertainty, external evidence, meaningful risk, multiple reviewers, or human-owned trade-off. | [Modes](#modes) |

Handoff is not a review mode. When a prior review already returned continue,
reuse its context pack instead of running a new review.

Reversibility lowers mode, not the review requirement.

## UX3

Review three connected dimensions:

| Flow | Core question | Source section |
|---|---| --- |
| User Flow | Who is the product for, who is affected, what are they trying to accomplish, and what cost or loss of control exists now? | [UX3](#ux3) |
| Evidence Flow | What signal was observed, where did it come from, what interpretation is being made, what counter-signal weakens it, and what decision impact follows? | [UX3](#ux3) |
| Business Flow | Who creates value, receives value, pays or decides, bears cost or risk, and can the exchange remain viable and legitimate? | [UX3](#ux3) |

Then inspect the intersections: Situated Understanding, Viable Learning,
and Sustainable Value Exchange.

## UX3 Decision Kernel

Apply the same reduction sequence to every submitted review:

1. Validate lane contracts and evidence references with `resources/scripts/check_review.py`.
2. Reduce to the worst verdict and deterministic weakest flow.
3. Gate uncertainty, risk, reversibility, and human-owned calls.
4. Emit one canonical verdict, next action, execution boundary, and stop conditions.

Do not average lanes. Human-owned calls are recorded through an accountable
decision record; they are not converted into a machine score.

Before handoff, name product organization, issuer, stakeholder effects, and
accountable owner. Missing issuer or owner blocks implementation.

## Operating Rules

- Do not implement during review.
- `verify` branch: verify authorizes only the exact proof_step and never
  general coding.
- `continue` branch: continue plus a valid context pack authorizes implementation.
- Every result states `may_do`, `must_not_do`, and `must_ask`.
- List disconfirming evidence before support.
- Do not average reviewer verdicts.
- The weakest flow controls the next action.
- External tools return evidence receipts, not verdicts.
- Evidence tier controls action size; confidence does not upgrade weak proof.
- Council mode requires independent reviews and a challenge round.
- If all independent Council reviews return continue, a valid
  resources/schemas/red-team-review.schema.json result must be attached to
  the Council input before synthesis.
- Designer taste is a human-owned input, never an evidence score.
- Incoming feedback must be classified before it changes direction or work.
- Human-owned decisions stay human-owned.

## Triggered Distilled Rules

Load only rules whose observable trigger applies. These families refine the
three Flow reviews; they do not create new verdicts or allow a strong Flow to
outvote a weak one.

| Situation | Required rule | Source section |
|---|---| --- |
| Mental-model mismatch, hidden capability, or misunderstood consequence | `ux3.rule.mental_model_alignment` | [Triggered Distilled Rules](#triggered-distilled-rules) |
| Cognitive or recovery burden across human, task, tool, and environment | `ux3.rule.human_factors_load` | [Triggered Distilled Rules](#triggered-distilled-rules) |
| Research claim, synthetic user, simulation, interview, survey, test, or analytics result | `ux3.rule.research_validity` | [Triggered Distilled Rules](#triggered-distilled-rules) |
| Value proposition, positioning, buyer, channel, or GTM path before fit is proven | `ux3.rule.value_market_path` | [Triggered Distilled Rules](#triggered-distilled-rules) |
| Agent plan or tool action that needs visibility, control, handoff, or recovery | `ux3.rule.human_agent_interaction` | [Triggered Distilled Rules](#triggered-distilled-rules) |
| Retention, personalization, or gamification that changes repeated behavior | `ux3.rule.motivation_ethics` | [Triggered Distilled Rules](#triggered-distilled-rules) |

## Outcome

| Verdict | Required next step | Source section |
|---|---| --- |
| continue | Create resources/templates/context-pack.md within the execution boundary. | [Outcome](#outcome) |
| verify | Run one proof step only. | [Outcome](#outcome) |
| stop_reframe | Stop execution and return a better product question. | [Outcome](#outcome) |

Use triggered rule IDs for detail instead of duplicating every rule in prompts.
Common triggers include ux3.rule.actor_boundary,
ux3.rule.problem_hypothesis, ux3.rule.evidence_separation,
ux3.rule.minimum_validated_proof, ux3.rule.feedback_classification, and
ux3.rule.human_judgment. The distilled trigger table above covers additional
rules that activate only when their specific product condition appears.

Use resources/docs/HARNESS.md for the handbook, resources/docs/OPERATING-PROTOCOL.md for the six-gate protocol,
resources/docs/CONTRACTS.md for output rules, and resources/examples/ for worked reviews.

## Anti-Patterns

Source: [Common Failure Modes](resources/docs/ANTI-PATTERNS.md#common-failure-modes).

| Anti-pattern | Problem | Fix |
|---|---|---|
| Rubber-stamp review | Every lane says continue and no counter-evidence appears. | Run the challenge round. |
| Evidence inflation | Quotes, opinions, logs, and payments are treated as equal proof. | Assign evidence tier and signal type. |
| Consensus theater | Reviewers repeat the same argument in different words. | Require objection_to and counter_evidence. |
| False precision | Confidence labels are averaged or treated as scores. | Use evidence tiers and the decision matrix. |
| Checklist substitution | All fields are filled but no decision changes. | State decision impact for every evidence receipt. |
| Context-pack leakage | A verify result becomes a broad build brief. | Allow only the proof step. |
| Tool authority | A connector or search result gives the verdict. | External tools return evidence receipts only. |
| Taste avoidance | Brand, dignity, or meaning is hidden inside a score. | Route to Human Judgment Gate. |
| Stale proof | Old evidence is reused after context changes. | Check freshness and re-verify expired sources. |
