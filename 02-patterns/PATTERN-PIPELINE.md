# Pattern: Pipeline (prompt chaining / sequential workflow)

**This file is a prompt for the agent.** A team is deciding how to structure their agent (a
search-and-extraction bot, or an experiment-guardian bot). The pipeline is the default
starting shape and usually the right one - so your job is to confirm it actually fits, then
help them develop it concretely: enumerate the stages, give each stage one job, and define
the deterministic gate between every pair of stages. Resist the urge to make it clever. A
fixed code path with one LLM call per stage and plain-code gates between them is the goal.

The teaching point: a pipeline is **a fixed code path with one LLM call per stage**. Between
stages sits a **deterministic gate** - plain code that validates, checks a schema, or retries.
The gate is NOT another agent: the moment a stage's output is checked by a model rather than by
plain code, you have reached for actor-critic (`../02-patterns/PATTERN-ACTOR-CRITIC.md`), not a
pipeline gate. Both are valid - just know which one you are building and why. A hybrid is often
the best of both: gate the STRUCTURE with deterministic code (schema valid, required keys
present, values parse, dates in range) and gate only the CONTENT a model is genuinely needed to
judge with a critic (is this extracted claim actually supported by the cited source? is this
"anomaly" real or noise?). One seam, two gates: cheap structural checks in code, an actor-critic
check on the narrow part that needs judgment. Each stage has its own prompt, its own tools, and
its own test.

Why it is the default shape - it serves five of the eight properties at once:

- **Context rot** - each stage gets a small, fresh window with only what it needs.
- **Specialization** - each stage has one focused prompt and few tools.
- **Observability** - every stage's input and output lands in a named file you can inspect.
- **Least privilege** - a stage holds only the tools its job needs (the extract stage cannot
  send email; only the final stage can act).
- **Testability** - you can test "extract fields" separately from "validate policy".

---

## How to run this session

1. **Confirm the shape fits.** A pipeline fits when the work has a fixed, known sequence of
   steps. If the team cannot say the steps in order yet, or the path branches on content,
   they may want routing (see `../02-patterns/PATTERN-ROUTING.md`) - check before committing.
2. **Enumerate the stages.** Get them to list the steps as an ordered sequence and name each
   one. Aim for one verb per stage.
3. **Give each stage ONE job.** For every stage, write: its input, its single job, the tools
   it needs (and only those), and its output. If a stage has two verbs, split it.
4. **Define the gate between every pair.** For each seam, write in plain code: what must be
   true for the work to advance, and what happens when it is not (retry / skip / stop). This
   is the part teams skip - do not let them.
5. **Place the handoffs on disk.** Each stage reads its input from, and writes its output to,
   a named file in the scaffold dir (see `../05-scaffold/SCAFFOLD.md`). Decide the exact
   paths now. The contents of each handoff follow `../03-handoff/DESIGN-HANDOFF.md`.
6. **Decide which stages even need an LLM.** Many "stages" are plain code (parse a PDF, query
   a DB, validate a schema). Use a model only where judgment is required.

---

## Worked examples (both team types)

**Search-extraction** - extract structured records from documents:

```text
01-read      (code)  PDF/patent -> plain text + page map
02-extract   (LLM)   text -> draft fields {claim, assignee, priority_date, ...}
  gate: every field present? each value has a source pointer (file:line)? else retry once.
03-validate  (LLM or code)  fields -> checked fields against schema + policy
  gate: schema valid AND dates parse AND assignee non-empty? else route to a review file.
04-write     (code)  checked fields -> structured record appended to the run output
```

**Experiment-guardian** - watch a run and act:

```text
01-poll      (code)  read latest sensor batch -> normalized readings
02-assess    (LLM)   readings -> {status: ok|warn|anomaly, why, evidence pointer}
  gate: status in the allowed set AND evidence points at a real log row? else mark needs-attention.
03-decide    (LLM)   assessment -> action {none | log | alert | escalate}
  gate: action allowed for this severity? alert only if assess said anomaly? else hold.
04-act       (code)  perform the one decided action; record it
```

Notice the gate is always plain code and the privileged step (write / act) is last and
narrow.

---

## Pushback to apply

- **"Stage 3 checks stage 2's work."** Good - but is the gate an LLM or plain code? Make the
  cheap structural checks (schema, presence, parse, allowed values) deterministic code. Save
  an LLM for judgment the gate genuinely cannot encode.
- **"What is the gate between stage 2 and stage 3?"** Make them answer in code, not vibes.
  "It checks the fields look right" is not a gate. "Every required key present and each value
  has a source pointer, else retry once then write to needs-review" is a gate.
- **"If stage 3 fails, then what?"** Force the choice: retry (how many times?), skip (and
  record why), or stop the run. Undefined failure handling is where pipelines silently
  corrupt their output.
- **Smuggling the transcript.** Ask: "What exactly does stage 2 pass to stage 3 - the fields,
  or the whole conversation that produced them?" Pass a small structured result, not the
  context. That is the handoff (`../03-handoff/DESIGN-HANDOFF.md`).
- **Which stage needs an LLM at all?** Challenge every stage: if plain code can do it
  deterministically, the model is just added cost and a place to go wrong.

---

## What to hand back

A stage table (input / job / tools / output / test), the gate definition for each seam in
plain code, and the exact handoff file paths in the scaffold dir. Then point them at
`../03-handoff/DESIGN-HANDOFF.md` to define what crosses each seam, and at
`../05-scaffold/SCAFFOLD.md` to lay out the directories.
