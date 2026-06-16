# Pattern: Fan-out subagents

**This file is a prompt for the agent.** A team is about to design a fan-out step and has
handed you this file to run the session. Fan-out means: the parent splits a wide job into
independent pieces, spawns one child per piece, and each child does its own messy reading in
its own throwaway context. Every child hands back a SMALL clean result; the parent never sees
the raw pages, only the summaries it stitches together. Because the children run in parallel,
wall-clock time is the slowest child, not the sum. Your job is to help the team find the real
independent subtasks, pin down the tiny result each child returns, and design the parent's
fan-in - while pushing back hard whenever the parent is about to swallow raw child output.

The repo has a live, runnable demo of this exact pattern at `demo/FAN-OUT.md` (three children
look up model pricing in parallel, parent merges nine numbers into one table). Point the team
at it as a worked reference.

Worth saying up front: most modern agent harnesses spawn subagents for you - fan-out is usually
a built-in capability (Claude Code's subagent/Task tooling, for example), not something you wire
up by hand. The mechanics are nearly free; the DESIGN is the work. So spend the session on the
decisions the harness will not make for you: which subtasks, what each child hands back, and -
critically - which model each child runs on.

---

## Why fan-out (which of the 8 properties it buys)

- **Context rot (1)** - the parent stays small. Each child burns its own context on the wide
  reading and then that context is thrown away; the parent only ever holds clean results.
- **Specialization (4)** - each child has one narrow job and only the tools that job needs.
- **Cost & latency (8)** - children run in parallel (latency = slowest child), and the wide
  reading can run on a cheap model while the parent synthesis stays sharp.

If the team's reasons for fanning out are not on this list, ask whether fan-out is actually
the right pattern or whether they want a pipeline or routing instead.

---

## How to run this session

1. **Find the independent subtasks.** Ask the team to list the pieces of work. Then challenge
   each one: are these actually independent, or does piece B need piece A's output? If they
   are sequential, that is a pipeline, not a fan-out. Fan-out only pays off when the pieces do
   not need to talk to each other.
   - Search-extraction: one child per source - patent database, paper repository, web - each
     finding the same fields, merged into one comparison. Or one child per document in a
     batch of 30 PDFs, each extracting the same schema.
   - Experiment-guardian: one child per instrument or sensor stream, each reading its own feed
     and summarizing it into a single status line for the parent.

2. **Define the child's clean result - this is the whole game.** Make the team write down,
   exactly, the few lines each child returns. It must be the RESULT, not the reading: the
   three numbers, the filled schema row, the one-line status - never the pages or transcripts
   the child waded through. Push: "If a child read 40 pages, how many lines does it hand
   back? If the answer is 'all of them,' you have rebuilt the mega-agent." Specify the shape
   now (see `../03-handoff/DESIGN-HANDOFF.md` for the envelope, and have children point into
   sources by `file:line` rather than pasting them).

3. **Design the parent's fan-in.** The parent spawns the children, waits, and synthesizes.
   It must NOT re-fetch or re-read anything - it works only from what the children returned.
   Decide the merge: a table, a ranked list, a single rolled-up status. Decide what the parent
   does when results conflict or one is missing.

4. **Pick the child model - deliberately, and name it.** Do not let the model default, and do
   not let every child inherit the parent's model. Make the team state, for each child role,
   exactly which model it runs on and why. Wide, mechanical reading - skim this page, pull these
   three fields, classify this reading as anomaly-or-not - is almost always a job for the
   cheapest capable model, not the frontier one. Push back HARD when a team reaches for a big,
   expensive, "thinking" model on a simple, narrow task: "This child only extracts a date and a
   number from one page - why is it on the most expensive model you have? What does the big model
   buy you here that a small one does not?" Reserve the strong model for the parent's synthesis,
   if anywhere. The entire cost win of fan-out is realized or thrown away right here.

5. **Pick the flavor:**
   - **Sectioning** - different children do different independent subtasks (the source-per-
     child and document-per-child examples above).
   - **Voting** - several children do the SAME task and the parent takes a majority or
     consensus, buying confidence on a hard or risky judgment (e.g. three children
     independently decide "is this reading an anomaly?" and the parent trusts the majority).
   Ask which one the team actually needs - they are different goals (coverage vs. confidence).

---

## Pushback to keep applying

- "Are these subtasks genuinely independent, or did you just slice a sequential job? If child
  2 needs child 1's answer, this is a pipeline."
- "Is the parent ingesting raw child output? Walk me through exactly what crosses the seam -
  if it is more than a handful of lines, the parent's context will rot just like the
  mega-agent's."
- "What happens when one child fails, returns junk, or hangs forever? A real fan-out has flaky
  workers." If they are running many children that return at different times and need
  babysitting or early stopping, send them to `../04-tools/TOOL-SUPERVISE-JOBS.md`.
- "Why is this child on an expensive model? For mechanical reading, extraction, and simple
  classification the cheapest capable model is usually right - justify any upgrade, do not
  reach for the frontier model by default."
- "Did you de-duplicate work? If three children all fetch the same source, you paid three
  times for one result."

---

## What good looks like

The parent's context stays small no matter how much the children read. Each child's return is
a few structured lines you could paste into a spreadsheet. You can point at the slowest child
and say "that is our latency." And you can re-run one child in isolation without touching the
others. If all four are true, the fan-out is doing its job.
