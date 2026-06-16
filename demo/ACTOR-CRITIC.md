# Live demo: actor - critic - judge (run with Haiku subagents)

**This file is a prompt for Claude.** During the talk the presenter will say something like
"run `demo/ACTOR-CRITIC.md`". When that happens, follow the steps below: run three Haiku
subagents in sequence (generator -> critic -> judge) on ONE task, then present the result
as a three-stage transcript.

The teaching point: **the same small model (Haiku 4.5) plays all three roles. The win comes
from structure, not from a bigger model.** A weak single answer becomes correct once a
fresh-context critic checks it and a judge reconciles.

---

## How to run it

1. Pick ONE task from the menu below (default: **Task A, strawberrraspberry**). If the
   presenter names a task or gives a new one, use that instead.
2. **Announce the plan first (for teaching).** Before spawning anything, print one short line
   on screen saying what is about to happen, e.g.: "I'll run the same small model (Haiku)
   three times in sequence on this question - a generator to answer it, a fresh critic to
   check that answer, and a judge to reconcile - then show the transcript." Then start.
3. Run the three stages, each as a separate `Agent` call with `model: "haiku"`:
   - **Generator** - answer the question (use the weak prompt to make a miss likely).
   - **Critic** - a FRESH subagent; paste the generator's answer in, tell it the answer
     may be wrong, have it recount/recheck from scratch.
   - **Judge** - a FRESH subagent; give it both answers, have it independently verify and
     pick the final answer.
   Stages are sequential (critic needs the generator's output; judge needs both). Within a
   stage there is only one call.
4. Present the output using the **Output format** below. Keep it tight - this is shown on
   screen while the presenter talks.

> Reliability note: the generator's miss is probabilistic. The weak prompt ("one word,
> don't think step by step") makes the error much more likely. If the generator happens to
> get it right, say so out loud (that is itself a fine teaching moment) - or re-run, or
> switch to **Task B (9.11 vs 9.9)**, which misses most reliably.

---

## Task menu

### Task A - letters in "strawberrraspberry"  (best showcase: the miss is buried in plausible work)

Correct answer: **6** r's (positions 3, 8, 9, 10, 16, 17).

- **Generator prompt:**
  > How many times does the letter "r" appear in the word "strawberrraspberry"? Answer with
  > the number and a quick letter-by-letter breakdown. Be fast, don't overthink it.

### Task B - which is bigger, 9.11 or 9.9  (most reliable miss; cleanest one-liner for a crowd)

Correct answer: **9.9**.

- **Generator prompt:**
  > Which is bigger: 9.11 or 9.9? Answer in one word, don't think step by step.

### Task C - letters "b" in "bubblegum bubble"  (backup; same flavour as A)

Correct answer: **6** b's (bubblegum = 3, bubble = 3).

- **Generator prompt:**
  > How many times does the letter "b" appear in the phrase "bubblegum bubble"? Answer with
  > the number and a quick breakdown. Be fast.

---

## Stage prompts (fill in {QUESTION} and {GENERATOR_ANSWER})

**Critic** (fresh Haiku):
> You are a critic. Another assistant was asked: "{QUESTION}"
>
> Its answer was:
>
> "{GENERATOR_ANSWER}"
>
> This answer may be wrong. Independently redo the task from scratch (count letter by letter
> / compare place value), then state whether the answer is correct. If it is wrong, give the
> correct answer and point out exactly where the error is.

**Judge** (fresh Haiku):
> You are a judge. The question was: "{QUESTION}"
>
> Two assistants disagree:
>
> - GENERATOR says: {GENERATOR_ANSWER}
> - CRITIC says: {CRITIC_ANSWER}
>
> Decide who is correct. Verify independently if needed, then give the single final answer
> and one sentence on why the loser was wrong.

---

## Output format (what to print on screen)

Render the run like this, substituting the real subagent text:

```text
TASK: <the question>

(1) GENERATOR  (Haiku)
    <generator's answer - show its breakdown, including the mistake>

(2) CRITIC     (fresh Haiku - "this may be wrong, redo it")
    <critic's independent result + where the generator went wrong>

(3) JUDGE      (fresh Haiku - sees both, verifies)
    FINAL: <final answer>   <one line on why the loser lost>
```

Then one punchline line, e.g.:
> Same model three times. The generator missed; the critic - fresh context, told to doubt -
> caught it; the judge confirmed. Structure beat the single shot.
