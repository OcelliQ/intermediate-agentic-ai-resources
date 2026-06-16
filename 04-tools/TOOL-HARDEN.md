# Harden a tool so it is safe to run in a loop

**This file is a prompt for the agent.** A team hands you this when they have a tool whose
*surface* and *interface* are already chosen (see `TOOL-SCOPE.md` and `TOOL-INTERFACE.md`)
and now needs to make it safe for an agent to call over and over, sometimes wrongly, often
on a retry. Your job is to brainstorm with them, push back on the unsafe parts, and then
help them actually wrap the capability into a CLI and write a smoke test. The teaching point:
an agent will call this tool in a loop, with imperfect arguments, and will retry on failure.
A tool that is fine to run once by a careful human is not yet safe to hand to an agent. The
work in this session is everything that stands between "works on the demo" and "survives the
loop."

This is phase 3 of the tool flow: **scope -> interface -> harden.** Interface decided *what
the call looks like*; here you decide *what happens when it goes wrong.*

---

## How to run this session

1. **Get the tool on the table.** Ask the team for the tool's current signature and what it
   does when called: what it reads, what it writes, what it touches in the outside world.
   Make them name the side effects explicitly. "It just extracts fields" - does it write a
   file? Overwrite one? "It just alerts" - does it page a human? Twice?
2. **Walk the five hardening questions below in order.** Each is a short conversation plus a
   concrete change to the tool. Do not let the team hand-wave a "yes, it's fine" - make them
   show you the behaviour or change the code.
3. **Wrap it into the chosen CLI** (the surface picked in `TOOL-SCOPE.md`). Help them write
   the actual command or script, with argument validation up front.
4. **Write a one-shot smoke test** (see the last section). This is the deliverable that
   proves the hardening is real and not just discussed.

---

## 1. Determinism - same input, same output

An agent that gets a different answer each call cannot be tested, cached, or trusted. Hunt
for hidden nondeterminism: unstable ordering, timestamps baked into output, "top N" that
reshuffles, a model call buried inside what should be a mechanical tool, reliance on wall
clock or network state that drifts.

- **Search-extraction:** `extract-fields <doc> <schema>` should return the *same* records in
  the *same order* for the same document and schema. If it returns fields in dict-iteration
  order or "whatever the parser found first," pin the ordering.
- **Experiment-guardian:** a `read-sensor` or `check-threshold` tool should report the value
  it actually read, not a smoothed or resampled number that changes between calls on the
  same data window.

> Pushback: "Call it twice on the same input - byte-for-byte the same output? If not, what
> changed, and can the agent tell the difference from a real change in the world?"

## 2. Good errors - the message is the recovery plan (backpressure)

When the tool fails, the agent only has the error string to decide what to do next. A bare
`Error: failed` or a 200-line stack trace are both useless. The error should say what went
wrong *and* what the agent can do about it: fix the argument, wait and retry, or give up and
hand back. This is backpressure - the tool telling the loop to slow down or change course.

- **Search-extraction:** unparseable PDF should return something like
  `ERROR unreadable: <doc> is image-only, no text layer; try the OCR tool or skip this doc`
  - not `Traceback ...`. A missing field should say which field and where it looked.
- **Experiment-guardian:** if the alert channel is down, return
  `ERROR channel-unreachable: Slack timed out after 5s; retry safe (idempotent), or escalate
  to phone` - a recoverable error the agent can act on, not a silent drop.

> Pushback: "Read only the error string - could the agent recover from this alone, with no
> other context? Does it say retry-safe or not?"

## 3. Idempotency - safe to retry anything with side effects

Agents retry. If a tool has a side effect - writes a file, sends a message, posts a record -
running it twice must not do the thing twice. Make every side-effecting call carry a key and
check for prior completion before acting.

- **Search-extraction:** write results keyed by document id into the scaffold dir
  (`.../03-extract/<doc-id>.json`). A re-run overwrites the same file deterministically; it
  does not append a second copy or spawn `<doc-id>-2.json`.
- **Experiment-guardian:** an `alert` tool must dedupe by **incident key**. Re-running the
  guardian after a crash must not fire a second page for the same incident. Store fired
  incident keys; on a repeat, return `already-alerted <key>, no-op` instead of paging again.

> Pushback: "The agent crashes mid-loop and re-runs the last three calls. What got
> double-done? Where is the key that makes the repeat a no-op?"

## 4. Least privilege - it can only touch what it must

A tool that *can* do damage eventually will, on a confused call. Scope it down to exactly the
capability it needs. Read-only by default; write only to its own stage dir; no broad
credentials it does not use.

- **Search-extraction:** the extractor reads documents and writes to its stage dir. It has no
  reason to delete sources or write outside the run dir - so do not give it the ability.
- **Experiment-guardian:** the guardian can *read* instrument state and *notify*. It must
  **not** be able to change setpoints, stop the rig, or write to the control system. If
  acting on the experiment is ever needed, that is a separate, separately-scoped tool with
  its own human gate.

> Pushback: "List everything this tool *can* touch versus everything it *must* touch. Close
> the gap. What is the blast radius if the agent calls it with garbage arguments?"

## 5. Poka-yoke - make the wrong call hard to make

Validate arguments at the front door and refuse ambiguous input loudly, before any side
effect. A tool that quietly does *something* with a malformed argument is worse than one that
rejects it. Prefer designs where the bad call is impossible to express.

- **Search-extraction:** reject an unknown schema name with the list of valid ones, rather
  than extracting against an empty schema and returning zero fields that look like a clean
  "nothing found."
- **Experiment-guardian:** require an explicit threshold and units; refuse a bare number.
  `check-threshold 80` is ambiguous - 80 what? Force `--metric temp_c --max 80`.

> Pushback: "What is the most plausible *wrong* call the agent will make? Does the tool catch
> it, or silently do the wrong thing? Can you make that wrong call un-expressible?"

---

## Build it: wrap the capability into the CLI

Now help the team turn the existing capability into the CLI surface chosen earlier. Keep the
wrapper thin: parse and validate arguments, call the underlying capability, format a terse
structured result (per `TOOL-INTERFACE.md`), and map failures to the good-error strings above.
Validation goes first, before any side effect.

## Build it: write a one-shot smoke test

The deliverable that proves the hardening is real. A tiny script or command the team can run
on demand that exercises:

- **the happy path** - one known input, assert the expected structured output (and that a
  second run produces byte-identical output: determinism + idempotency in one check); and
- **one error path** - one known-bad input, assert a clean, recoverable error string rather
  than a crash or a silent success.

- **Search-extraction:** run `extract-fields fixtures/sample.pdf basic-schema` twice, diff the
  two outputs (must be identical), then run it on `fixtures/image-only.pdf` and assert the
  `unreadable` error.
- **Experiment-guardian:** call `alert --incident TEST-1` twice and assert the second is a
  `already-alerted` no-op, then call with the channel pointed at a dead endpoint and assert
  the `channel-unreachable, retry safe` error.

> Closing pushback: "If you handed this tool to a tired agent at 3am that retries everything
> twice and gets half its arguments wrong - what breaks? Your smoke test should already
> answer that."
