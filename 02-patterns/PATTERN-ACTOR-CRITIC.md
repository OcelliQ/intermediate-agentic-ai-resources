# Pattern: Actor-critic / generate-then-verify

**This file is a prompt for the agent.** A team is about to design an actor-critic step and has
handed you this file to run the session. The pattern: a generator does the task, a critic with
a FRESH context tries to find what is wrong with the generator's answer, and a judge sees both
the draft and the critique and decides the final output. The surprising part - worth saying out
loud to the team - is that this works even when all three roles are the SAME model. The win is
structural: a fresh context told "this may be wrong, find the error" catches mistakes that the
generator, invested in its own answer, glides past. Your job is to help the team define the
three roles crisply and to push back hard on the two ways this pattern quietly fails: a critic
that is not really fresh, and a judge that just rubber-stamps.

The repo has a live, runnable demo at `demo/ACTOR-CRITIC.md` (the same Haiku model plays all
three roles; the generator miscounts, the fresh critic catches it, the judge confirms). Point
the team at it as a worked reference.

---

## Why actor-critic (which of the 8 properties it buys)

- **Credulity (2)** - a dedicated critic is the antidote to an agent that trusts its own first
  answer (or trusts a poisoned source). Someone whose only job is doubt looks where the
  generator did not.
- **Testability (7)** - the critique is an explicit, inspectable artifact. You can read why an
  answer was rejected, log it, and measure how often the critic catches real errors.

---

## How to run this session

1. **Define the generator's task.** What does it produce, and in what shape? Keep this the same
   focused job it would have had alone - the critic and judge wrap around it, they do not change
   it.
   - Search-extraction: the generator extracts the structured fields or claims from a patent or
     paper (inventor, priority date, the specific numeric claim).
   - Experiment-guardian: the generator interprets a result or flags an anomaly ("the reaction
     temperature spiked, this looks like runaway").

2. **Define the critic's narrow job - and make it FRESH.** The critic gets a brand-new context.
   It is told the answer MAY be wrong and that its only job is to find what is wrong - not to
   agree, not to praise. Crucially, decide what the critic checks AGAINST: it should verify
   against the actual source, not against the generator's reasoning.
   - Search-extraction: re-check each extracted field against the source text. Did the
     generator hallucinate a value, or misattribute one claim's number to another? The critic
     should point at the exact `file:line` for each field (see
     `../03-handoff/DESIGN-HANDOFF.md` on citations and line-pointers - the critic's findings
     are far stronger when they cite the source rather than re-asserting from memory).
   - Experiment-guardian: challenge the statistics, the confounds, and the false-alarm risk.
     "Is this spike outside normal noise, or within it? Could a sensor glitch explain it before
     we wake someone at 3am?"

3. **Define the judge.** The judge sees the draft AND the critique and decides the final
   output - it does not loop the generator and critic against each other forever. It reconciles:
   keep what is verifiable, drop what the critic refuted, and produce one answer. Decide now
   whether the critic and judge are the same role (critic returns the corrected answer directly)
   or separate (a third agent reconciles two disagreeing inputs). Separate is stronger when the
   stakes are high or the critic itself might overreach.

---

## Pushback to keep applying

- "Does the critic actually have a FRESH context, or is it the same agent in the same
  conversation agreeing with itself? A critic that has already seen the generator 'reason' to
  the answer will rationalize it. Spawn it clean."
- "Is the judge reconciling, or just rubber-stamping? If the judge always sides with the
  generator, you have added cost and no safety. Make it independently verify the disputed
  point."
- "What concretely does the critic check against - the source text? the raw data? - or just its
  own gut? Vague critique is theater. Pin it to the evidence."
- "Are you spinning? Generate-critique-regenerate-critique forever is a failure mode. The judge
  decides once and stops."

---

## What good looks like

Three roles, each with one job and the critic in a genuinely fresh context. The critique is a
concrete artifact that points at specific evidence, not a vibe. The judge produces exactly one
final answer, reconciling the two, and the loop terminates. Run it on a case where the
generator alone gets it wrong, and the structure catches the error - that is the whole point,
and it is worth demonstrating to the team on their own data.
