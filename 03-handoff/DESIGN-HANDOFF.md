# Design a handoff between two agents

**This file is a prompt for the agent.** A team has handed you this file to start a working
session. Your job is not to lecture - it is to help them design ONE concrete handoff contract for
ONE real seam between two of their agents, push back hard when the design is vague, and leave them
with a file they can drop into their pipeline. A handoff is a small *envelope*: a payload plus just
enough metadata for the next agent to act - NEVER the conversation that produced it. Teams get this
wrong by passing too much (the whole transcript) or too little (a bare string with no provenance).

The teaching point: **the handoff is the API between two agents. Treat it like one** - testable,
observable, and reasoned about without ever reading the insides of either agent. Four principles from
the deck ladder under every pushback below: (1) pass just enough information, (2) one owner at a
time, (3) make the seam observable (a named file), (4) test/enforce the interface (a gate rejects a
malformed handoff instead of passing it downstream).

---

## How to run this session

1. **Find the seam.** Name the two agents and the single point where control passes. If they can't
   say it in one sentence ("the searcher hands the extractor a list of candidate documents"), the
   seam is not yet real - make them draw it first.
2. **Name the direction and owner.** Who produces, who consumes, who owns the task *after* the
   handoff. Exactly one owner on each side of the line.
3. **Walk the envelope checklist**, item by item, filling in their actual case. Do not let them
   hand-wave "we'll just pass the data" - make each field concrete.
4. **Pick the form**: text, structured, or both (structured at machine seams, text where a human
   reads the result). See the fork-on-form section.
5. **Write the gate.** Where is the handoff validated, and what happens when it fails? A handoff
   with no gate is a hope, not a contract.
6. **Produce the artifact** - one real handoff file in the right scaffold directory (see
   ../05-scaffold/SCAFFOLD.md), plus the gate that checks it.

Throughout, default to *less in the envelope, more in the directory*: the scaffold already carries
much of what teams are tempted to stuff into the payload.

---

## The envelope checklist

The spine of the session. Fill in all eight for the team's seam; if one genuinely does not apply,
make them say why out loud.

### 1. The payload - the deliverable, and nothing else

The actual result the next agent needs: a named document/section (text) or a typed record
(structured), e.g. `{patent_id, claim_number, claim_text_ref}` or `{run_id, metric, value,
threshold, breached}`. *Pushback: are you passing the result or the conversation? If the producer
thought out loud for forty turns, only the conclusion crosses the seam.* e.g. the candidate-document
IDs, not the searcher's reasoning; the breach decision, not the whole sensor stream.

### 2. Keys / provenance - what this result is *about*

Only the IDs the receiver needs to act (query / patent / run ID). Most provenance is already carried
by the scaffold directory (which run, which stage), so do not re-encode it in prose. *Pushback:
which of these IDs does the receiver actually use, versus which are implied by the folder this file
lives in? Drop the latter.*

### 3. Status & confidence - did it actually work?

One of succeeded / partial / failed, a confidence signal, and an explicit "could not resolve X" -
the escape hatch, not a silent guess. A producer that always reports success cannot be trusted.
*Pushback: when the searcher finds nothing, or the watcher is unsure a spike is real, what does this
field say?* e.g. `status: partial, unresolved: ["claim 7 - OCR garbled"]`; or `status: ok,
confidence: low, note: "single sensor, no corroboration"`.

### 4. Gaps, assumptions, open questions - what the receiver must now handle

What the producer did NOT do, and any assumption the receiver is now inheriting - so it can check
the premise instead of trusting it. *Pushback: what assumption, if wrong, breaks the receiver? Write
it down.* e.g. "English-language patents only; 3 non-English skipped"; "calibration from run-02
assumed valid, not re-verified".

### 5. Schema + version (structured) / expected shape (text)

The contract this conforms to, named explicitly - a schema name + version (`extraction-result/v2`)
for structured, the expected sections for text. The gate validates against this, so it must be
named. *Pushback: when you change the format next week, how does the receiver know? No version means
silent drift becomes a silent bug.*

### 6. Ownership / next action - the explicit transfer of control

A plain statement: "I am done. You now own this task. Here is what you do next." This enforces one
owner at a time. *Pushback: after this handoff, can both agents think they own the task? If so, they
fight or both go idle. Who holds it - exactly one?*

### 7. Audit metadata - who, when, which agent, which attempt

Producer agent, model, timestamp, attempt number - most of it reflected by the versioned filename
and run directory, so lean on the scaffold rather than duplicating it. *Pushback: if this handoff is
wrong, can you tell which agent, model, and attempt produced it - from the file alone or from where
it sits?*

### 8. Reference, do not copy - point into sources, never paste them

The token-efficiency rule, and the one teams most often get wrong. **Do not paste an eighty-page
patent or a full log into the envelope** - point to it with a relative path plus a line range
(`sources/patent_US123.txt:412-419`), a page/paragraph anchor, or a stable ID plus offset; the
receiver pulls full context only if it needs to. When a claim rests on the source, quote directly
but minimally: the exact short quote *plus* the pointer, so the next agent and a human auditor can
verify in context without trusting a paraphrase. Structured form is a citation object, not a blob:

```json
{ "source_id": "US7654321B2", "path": "sources/patent_US123.txt",
  "line_start": 412, "line_end": 419,
  "quote": "a heat sink thermally coupled to the said substrate" }
```

Search-extraction points at `file:line` in the document; experiment-guardian points at a log line /
timestamp / data row (`logs/run-07.csv:2026-06-16T14:32:01`), not a dump of the whole log.
*Pushback: is the receiver getting a five-line citation or a fifty-page paste? Can a human confirm
each claim by following the pointer? If the source is regenerated on a re-run, does the pointer
still resolve - and if not, what anchors it (a stable ID, a hash, a header line)?*

---

## Fork on form: text vs structured

Most seams want one; some want both - a structured record for the next agent plus a human-readable
summary for the audit trail. Decide deliberately.

**Text handoff** - a named markdown file: a short header block (the metadata) followed by the body
(the payload). The gate is a lightweight check (required header fields present, every claim carries
a pointer).

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

> Pushback: "How does the receiver know this is the *final* version, not a draft from a crashed
> attempt? The filename and the status field together should answer that."

**Structured handoff** - a JSON or typed record validated against a schema; the metadata become
fields. The gate is a **real validator that rejects a malformed handoff**, not one that passes a
half-filled record downstream.

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

**Where do you validate, and what happens on failure?** Validate at the gate between stages (plain
code, not another agent). On failure the choice is crash the run, recover with a default (dangerous,
it hides the problem), or route to a fix path (retry, or a human). **Default: reject and stop**,
write the bad handoff to the run directory so it is observable, and surface the validation error - a
silently recovered bad handoff is the worst outcome. *Pushback: what does the receiver do when a
required field is missing - crash, or recover? If it recovers, you turned a contract violation into
a silent guess. Where is that decision written down?*

---

## Land the artifact

1. Pick ONE seam in the team's actual pipeline.
2. Write its handoff contract - schema (named + versioned), an example envelope, and the gate that
   validates it; do both a text and a structured variant if both are warranted.
3. Save it into the right scaffold directory (../05-scaffold/SCAFFOLD.md) so it is observable and
   re-runnable - the handoff file lives between the producing and consuming stage directories.
4. Read it back and ask: **"If I handed this file to a stranger, could they tell what crossed the
   seam, whether it succeeded, and where every claim came from - without reading either agent?"** If
   not, the envelope is not done.
