# Tool design, phase 2: the schema is the prompt

**This file is a prompt for the agent.** The team has decided what tool to build and what
surface it takes (phase 1, `TOOL-SCOPE.md`). Now design the contract the agent sees: the
name, the description, the parameters, and - the part everyone underestimates - the *return
shape*. The schema is not paperwork around the tool; the schema IS the prompt. It is the only
thing a fresh agent reads to decide whether to call the tool, how to call it, and how to use
what comes back. A tool is UX for the model. Design it like one.

This is phase 2 of three: **scope -> interface -> harden**. Phase 3 (`TOOL-HARDEN.md`) makes
the tool safe to run repeatedly; here we make it *legible* and *cheap*.

---

## How to run this session

1. **Name and describe it as if the agent has never seen it.** Draft the tool/command name
   and a one or two line description. The description must say *when* to use it and *when not
   to*. A fresh agent with only this line should call it at the right moment and skip it at
   the wrong one.

2. **Design the parameters.** For each: name, type, and whether it is required. Names should
   make a wrong call obvious - `max_results` not `n`, `patent_id` not `arg2`. Prefer a few
   required parameters with good defaults over a wall of optional knobs.

3. **Design the return shape - this is the main event.** Decide the *minimum* structured
   result the next step needs, and return only that. Then decide what does NOT come back over
   stdout but instead gets written to a file the agent can open later. Big payloads (full
   documents, raw telemetry, long logs) belong on disk in the run directory; the tool returns
   a *pointer* to them, not their contents. (This is what lets later handoffs cite
   `source.txt:412-419` instead of pasting the whole document - see the scaffold and handoff
   prompts.)

4. **Walk the cold-read test.** Hand yourself only the schema you just wrote, with no other
   context, and ask: would I know when to call this, and how to read the result? Fix whatever
   fails that test.

5. **Record and hand off.** Capture the name, parameters, and return shape, then move to
   `TOOL-HARDEN.md`.

---

## Pushback to give

- "Hand a fresh agent ONLY this schema. Would it know WHEN to call this, and HOW to read what
  comes back? If not, the description is doing too little."
- "Does the return dump everything, or the minimum the next step needs? What in here is the
  agent going to read once and never use?"
- "Where does this waste tokens? Should that big field be a file path instead of inline text?"
- "Do the names remove ambiguity, or invite a wrong call? Would a different reasonable agent
  read `count` as 'how many to return' or 'how many exist'?"
- "What happens when there are zero results, or ten thousand? Does the shape still make sense?"

---

## Both team types

- **Search-and-extraction:** a `patent-search` CLI should return a compact list of
  `{id, title, score}` - not full texts. The agent scans the list cheaply, then calls a
  separate `patent-get <id>` only for the few it actually needs, which writes the full text to
  a file and returns its path. The search context stays small; the heavy reading is deferred
  and lands on disk.
- **Experiment-guardian:** a `get-reading` should return
  `{sensor, value, unit, timestamp, status}` - not a wall of telemetry. If the team needs the
  full trace, that is a separate call that writes the trace to a file and returns the path,
  so the watching loop is not flooded with numbers it will not read.

---

## What this phase produces

A tool contract the team carries into phase 3:

```text
Name:          <command or tool name>
Description:   <when to use / when not to>
Parameters:    <name: type (required?)> ...
Returns:       <the minimal structured result>
Writes to disk: <what large output goes to a file, and what the return points at>
```
