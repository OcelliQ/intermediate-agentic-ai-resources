# Design a handoff between two agents

**This file is a prompt for the agent.** A team has handed you this file to start a working
session. Your job is not to lecture them - it is to help them design ONE concrete handoff
contract for ONE real seam between two of their agents, push back hard when their design is
vague, and leave them with a file they can actually drop into their pipeline. A handoff is a
small *envelope*: a payload plus just enough metadata for the next agent to act. It is NEVER
the conversation that produced it. Most teams get this wrong by passing too much (the whole
transcript) or too little (a bare string with no provenance). Your job is to find the seam,
size the envelope, and make it enforceable.

The teaching point you are driving toward: **the handoff is the API between two agents. Treat
it like one.** A good seam can be tested, logged, and reasoned about without ever reading the
insides of either agent.

---

## Before you start: the four principles this session serves

These come from the deck. Keep them in view; every pushback question below ladders up to one.

1. **Pass just enough information** - a small, structured result, not the whole conversation.
2. **One owner at a time** - control transfers explicitly; two agents never own one task.
3. **Make the seam observable** - every handoff is a named file you can read after the fact.
4. **Test/enforce the interface** - a gate validates the handoff; a malformed one is rejected,
   not silently passed downstream.

---

## How to run this session

1. **Find the seam.** Ask the team to name the two agents and the single point where control
   passes from one to the other. If they can't name it in one sentence ("the searcher hands the
   extractor a list of candidate documents"), that is your first finding - the seam is not yet
   real. Make them draw it before anything else.
2. **Name the direction and the owner.** Who produces, who consumes, and who owns the task
   *after* the handoff. There is exactly one owner on each side of the line.
3. **Walk the envelope checklist below**, item by item, filling in their actual case. Resist
   the urge to let them hand-wave "we'll just pass the data" - make each field concrete.
4. **Pick the form**: text envelope, structured envelope, or both. Most pipelines need
   structured at machine seams and text where a human reads the result. Use the fork-on-form
   section below.
5. **Write the gate.** Decide where the handoff is validated and what happens when validation
   fails. A handoff with no gate is a hope, not a contract.
6. **Produce the artifact.** End the session by writing one real handoff file into the right
   scaffold directory (see ../05-scaffold/SCAFFOLD.md), plus the gate that checks it.

Throughout, default to *less in the envelope, more in the directory*. The scaffold already
carries a lot of what teams are tempted to stuff into the payload.

---

## The envelope checklist

This is the spine of the session. For the team's seam, fill in all eight. Skip nothing; if an
item genuinely does not apply, make them say why out loud.

### 1. The payload - the deliverable, and nothing else

The actual result the next agent needs. For a **text** handoff this is a named document or
section ("the extracted abstract", "the anomaly report"). For a **structured** handoff it is a
typed record (a list of `{patent_id, claim_number, claim_text_ref}`, or a
`{run_id, metric, value, threshold, breached}` reading).

> Pushback: "Are you passing the *result* or the *conversation*? If the producing agent thought
> out loud for forty turns, none of that crosses the seam - only the conclusion does."

- Search-extraction: the payload is the list of candidate documents with their IDs, not the
  searcher's reasoning about why it picked them.
- Experiment-guardian: the payload is the breach decision and the reading that triggered it,
  not the full sensor stream the watcher scanned.

### 2. Keys / provenance - what this result is *about*

The IDs the receiver needs to act: query ID, source / patent / paper ID, experiment ID + run
ID. **Most provenance is already carried by the scaffold directory** (which run, which stage),
so do not re-encode it in prose. Put in the envelope only the keys the receiver actually needs
to do its job - usually a handful of IDs that point back into the run.

> Pushback: "Which of these IDs does the receiver *use*, versus which are already implied by
> the folder this file lives in? Drop the ones the directory already tells you."

### 3. Status & confidence - did it actually work?

One of fully succeeded / partial / failed, plus a confidence or completeness signal, plus an
explicit **"could not resolve X"**. This is the escape hatch surfacing instead of a silent
guess. A producer that always reports success is a producer you cannot trust.

> Pushback: "When the searcher finds nothing, what does this field say? When the watcher is
> unsure whether a spike is real, does it pass a confident reading or flag the uncertainty?"

- Search-extraction: `status: partial` with `unresolved: ["claim 7 - OCR garbled, see ref"]`.
- Experiment-guardian: `status: ok` with `confidence: low, note: "single sensor, no corroboration"`.

### 4. Gaps, assumptions, open questions - what the receiver must now handle

What the producer did NOT do, and any assumption it made that the receiver is now inheriting.
This prevents the receiver from silently building on a wrong premise.

> Pushback: "What assumption did the producer make that, if wrong, breaks the receiver? Write it
> down so the receiver can check it instead of trusting it."

- Search-extraction: "Assumed English-language patents only; 3 candidates were non-English and
  skipped."
- Experiment-guardian: "Assumed the calibration from run-02 still holds; not re-verified this run."

### 5. Schema + version (structured) / expected shape (text)

Which contract this handoff conforms to, named explicitly. For structured, a schema name and a
version (`extraction-result/v2`). For text, the expected sections. This is principle 4 made
concrete: the gate validates *against this*, so it has to be named.

> Pushback: "When you change this format next week, how does the receiver know? If there is no
> version, a silent drift becomes a silent bug."

### 6. Ownership / next action - the explicit handoff of control

A plain statement: "I am done. You now own this task. Here is what you do next." This enforces
*one owner at a time*. Ownership is also reflected by which stage directory wrote the file, but
state it anyway so the transfer is unambiguous.

> Pushback: "After this handoff, can both agents think they own the task? If so, you have a seam
> where two agents fight or both go idle. Who holds it - exactly one?"

### 7. Audit metadata - who, when, which agent, which attempt

Producer agent name, model, timestamp, attempt number. Much of this is reflected by the
versioned filename and the run directory, so lean on the scaffold (see ../05-scaffold/SCAFFOLD.md)
rather than duplicating it. Keep in the envelope only what a reader needs without `ls`-ing the
folder.

> Pushback: "If this handoff is wrong, can you tell which agent, which model, and which attempt
> produced it - from the file alone or from where it sits?"

### 8. Reference, do not copy - point into sources, never paste them

This is the token-efficiency rule, and it is the one teams most often get wrong. **Do not paste
an eighty-page patent or a full experiment log into the envelope.** Point to it:

- A relative path into the scaffold dir **plus a line range** - `sources/patent_US123.txt:412-419`.
- Or page / paragraph anchors, or a stable ID plus offset.
- The receiver pulls the full context only if it needs to (lazy loading). The envelope stays small.

When a claim rests on the source, **quote directly but minimally**: include the exact short
quote that matters *and* the pointer beside it. That way the next agent - and a human auditor -
can verify the claim in context without trusting a paraphrase.

Structured form: a citation object, not a blob:

```json
{ "source_id": "US7654321B2", "path": "sources/patent_US123.txt",
  "line_start": 412, "line_end": 419,
  "quote": "a heat sink thermally coupled to the said substrate" }
```

- Search-extraction: point at `file:line` inside the document you extracted from.
- Experiment-guardian: point at a **log line / timestamp / data row** -
  `logs/run-07.csv:2026-06-16T14:32:01` - not a dump of the whole run log.

> Pushback: "Is the receiver getting a five-line citation or a fifty-page paste? Can a human
> confirm each claim by *following the pointer*? If the source file is regenerated on a re-run,
> does your pointer still resolve - and if not, what anchors it (a stable ID, a hash, a header
> line) instead of a fragile line number?"

---

## Fork on form: text vs structured

Most seams want one or the other. Some want both - a structured record for the next agent plus
a human-readable summary for the audit trail. Decide deliberately.

### Text handoff - a named markdown file

A short **header block** (the metadata items above) followed by the **body** (the payload).
Human-readable, loosely structured. The gate is a lightweight check: does the header have the
required fields, does every claim in the body carry a pointer.

Example - an extraction handoff dropped at a seam:

```markdown
# Handoff: candidate claims for review
status: partial
confidence: medium
schema: extraction-result/v2
owner_next: claim-validator
unresolved:
  - claim 7 - OCR garbled, see ref below

## Claims
- **Independent claim 1** names "a heat sink thermally coupled to the said substrate".
  ref: sources/patent_US123.txt:412-419
- **Claim 7** could not be parsed cleanly (OCR noise).
  ref: sources/patent_US123.txt:980-991
```

> Pushback: "How does the receiver know this is the *final* version and not a draft left behind
> by a crashed attempt? The filename and the status field together should answer that."

### Structured handoff - a validated record

A JSON or typed record validated against a schema. The metadata become fields. The gate is a
**real validator that rejects a malformed handoff** - it does not pass a half-filled record
downstream.

Example - a guardian breach handoff:

```json
{
  "schema": "breach-report/v1",
  "run_id": "run-2026-06-16T14-30-00",
  "status": "ok",
  "confidence": "high",
  "owner_next": "alert-dispatcher",
  "breached": true,
  "metric": "reactor_temp_c",
  "value": 91.4,
  "threshold": 85.0,
  "citations": [
    { "source_id": "sensor-A", "path": "logs/run-07.csv",
      "line_start": 4120, "line_end": 4120,
      "quote": "2026-06-16T14:32:01,reactor_temp_c,91.4" }
  ],
  "assumptions": ["calibration from run-02 assumed valid; not re-verified"]
}
```

**Where do you validate, and what happens on failure?** Make the team answer this explicitly:

- Validate at the *gate between stages* - plain code, not another agent.
- On failure, the choice is: crash the run (loud, good for a pipeline you watch), recover with a
  default (dangerous - hides the problem), or route to a fix path (a retry, or a human).
- Default recommendation: **reject and stop**, write the bad handoff to the run directory so it
  is observable, and surface the validation error. A silently recovered bad handoff is the worst
  outcome.

> Pushback: "What does the receiver do when a required field is missing - crash, or recover? If
> it recovers, you have just turned a contract violation into a silent guess. Where is that
> decision written down?"

---

## Land the artifact

Close the session by producing something real:

1. Pick ONE seam in the team's actual pipeline.
2. Write the handoff contract for it - the schema (named + versioned), an example envelope, and
   the gate that validates it. Do both a text and a structured variant if both are warranted.
3. Save it into the right scaffold directory so it is observable and re-runnable. The handoff
   file lives between the producing and consuming stage directories - see
   ../05-scaffold/SCAFFOLD.md for the exact layout
   (`pipeline-<name>/run-<datetime>/<stage-number>-<stage-name>/<files>`).
4. Read it back and ask the closing question: **"If I handed this file to a stranger, could they
   tell what crossed the seam, whether it succeeded, and where every claim came from - without
   reading either agent?"** If not, the envelope is not done.
