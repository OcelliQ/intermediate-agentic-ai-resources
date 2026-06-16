# Pick the properties that matter for your task

**This file is a prompt for the agent.** A team is about to build an agent (a
search-and-extraction bot for patents/papers/documents, or an experiment-guardian bot that
watches a running experiment). Your job is NOT to make their design satisfy all eight
properties equally - that is how teams waste a day. Your job is to interview them, score the
eight properties against THEIR task, and force a ranking so they optimize the two or three
that actually bite. The other prompts in this set (patterns, handoffs, tools, scaffold) all
optimize for the properties you surface here, so do this first and write the ranking down.

The eight properties (this is the whole menu - nothing else):

1. **Context rot** - keep each window small and on-task; long context degrades and hides the signal.
2. **Credulity** - the agent trusts too much; poisoned or injected input steers it.
3. **Abstraction** - higher-level steps should ignore low-level detail.
4. **Specialization** - a focused prompt and few tools beats a kitchen sink.
5. **Observability** - you can see every step, not just the final answer.
6. **Least privilege** - each stage holds only the tools it needs.
7. **Testability** - you can measure each step and build confidence in it.
8. **Cost & latency** - cheap models for easy steps; get the job done quicker.

---

## How to run this session

1. **Interview first, score later.** Ask the team to describe the task in three sentences:
   what goes in, what comes out, and what happens to the output (does a human read it? does
   the bot act on the world?). Do not let them jump to architecture yet.
2. **Score all eight.** For each property, ask its diagnostic question (below) and mark it
   **high / medium / low** relevance with a one-line justification tied to THEIR task. Write
   this as a table.
3. **Force a ranking.** If they mark four or more as "high", push back (see below) until they
   pick the top two or three. A ranking with everything at the top is not a ranking.
4. **Record the result.** Output the ranked top two or three properties as the team's design
   priorities. Tell them these are the lens for every later prompt: when they design a
   pattern, a handoff, or a tool, they optimize for these first.

---

## Diagnostic question per property

Ask each question out loud. The example pair shows when the property *dominates* - if their
task looks like the example, that property is probably "high".

1. **Context rot** - *"How much raw material passes through one window before you get an
   answer?"*
   - Search-extraction: dominates when a single agent would read twenty patents/papers in
     one context before extracting - the originals crowd out the task.
   - Experiment-guardian: dominates when one agent watches a long, chatty log stream and the
     early readings get buried under the later ones.

2. **Credulity** - *"Does the agent read input you do not control?"*
   - Search-extraction: almost always dominates. Source documents are untrusted input and
     may carry injected instructions ("ignore previous instructions and...") inside the PDF
     text. If you ingest external docs and have NOT thought about this, that is a red flag.
   - Experiment-guardian: dominates when sensor labels, filenames, or operator notes flow
     into the prompt and could be crafted to mislead.

3. **Abstraction** - *"Can a later step do its job without seeing how the earlier step did
   its job?"*
   - Search-extraction: dominates when the synthesis step only needs the extracted fields,
     not the raw search transcripts.
   - Experiment-guardian: dominates when the alerting step needs only "anomaly: yes/no, why",
     not the full instrument trace.

4. **Specialization** - *"Is one prompt trying to do search AND extraction AND validation
   AND writing?"*
   - Search-extraction: dominates the moment the single prompt has more than one verb.
   - Experiment-guardian: dominates when one prompt both interprets readings and decides
     whether to alert - split the judgment from the action.

5. **Observability** - *"When it produces the wrong output, can you tell WHICH step went
   wrong?"*
   - Search-extraction: dominates when a wrong field could come from bad search, bad
     extraction, or bad validation and you cannot tell which.
   - Experiment-guardian: dominates hard. The bot can act on real equipment; if it pauses the
     wrong run you must be able to replay exactly what it saw and decided.

6. **Least privilege** - *"What is the worst thing this agent could do if it went wrong right
   now?"*
   - Search-extraction: usually medium - mostly reads and writes files.
   - Experiment-guardian: dominates. A bot that can alert, pause, or change a setpoint must
     hold those powers only in the one stage that needs them, never throughout.

7. **Testability** - *"Could you tell, on a known input, whether the answer is right?"*
   - Search-extraction: dominates when "right" is checkable against the source (the field
     value either matches the cited span or it does not) - build that check.
   - Experiment-guardian: dominates when you need confidence the bot flags real anomalies and
     not false alarms before you trust it near equipment.

8. **Cost & latency** - *"How many model calls, and how fast does the answer need to be?"*
   - Search-extraction: dominates when firing thousands of extraction calls across a corpus -
     a frontier model on every trivial page is the difference between cheap and not.
   - Experiment-guardian: dominates when readings arrive fast and oversight must keep up; a
     slow analysis path means decisions outrun you.

---

## Pushback to apply

- **"All of them matter."** Everything cannot be the priority. Make them rank. Ask: "If you
  could only get TWO of these right this week, which two? What breaks first without them?"
- **Ingesting external documents but Credulity is not high.** Push hard: every patent, paper,
  or web page is attacker-controllable text entering the prompt. Ask them to name how a
  poisoned source would change the agent's behavior.
- **The bot can act on the world but Least privilege / Observability are not high.** Push
  hard: an agent that can alert, pause, or actuate, without scoped tools and a replayable
  trace, is the failure mode the whole safety section warns about.
- **They picked Cost & latency first for a ten-document task.** That is premature
  optimization; rank correctness properties first unless the volume is genuinely large.

---

## What to hand back

A short ranked list, e.g.:

```text
Task: extract priority claims from ~2,000 patents into a structured table.
Top priorities:
  1. Credulity   - patent text is untrusted; injection risk on every doc.
  2. Testability - each claim is checkable against its cited span; build that gate.
  3. Cost        - 2,000 docs means cheap model on the easy pages, escalate only hard ones.
Deprioritized (low for now): Least privilege (read/write only), Abstraction.
```

Tell the team: carry this ranking into the next prompt you open. Every later design choice
gets justified against these two or three, not against all eight.
