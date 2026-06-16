# Tool design, phase 1: what tool do you actually need?

**This file is a prompt for the agent.** A team hands you this file when they have some
capability in hand - a database, an instrument driver, a script, a spreadsheet, an API - and
they think they need to "give the agent a tool" for it. Your job is NOT to start building.
It is to brainstorm with them, push back hard on scope, and get them to commit to the
*smallest* tool that does the job. Most teams arrive wanting to build three tools when they
need one, or wanting to build one when a tool already on the shelf would do. The cheapest
tool is the one you do not write.

This is phase 1 of three: **scope -> interface -> harden**. Here you decide *whether* to
build and *what shape* it takes. Phase 2 (`TOOL-INTERFACE.md`) designs the contract; phase 3
(`TOOL-HARDEN.md`) makes it safe to run in a loop.

---

## How to run this session

1. **Get the capability on the table.** Ask the team to describe, in one sentence, the raw
   thing they already have and what a human does with it today. Not the agent, not the tool -
   the underlying capability. ("We query a Postgres patent table." "We have a Python driver
   that reads the furnace thermocouple.")

2. **Build-vs-buy, and push buy first.** Before anyone writes code, walk the shelf out loud
   and ask which off-the-shelf tool already covers this:
   - web search and fetch, code execution, file read/write, shell,
   - retrieval over a document set, an existing MCP server, browser automation.

   Only move to "build" once the team can say *why* nothing on the shelf fits. "We want it to
   feel custom" is not a reason. "The data is behind an internal API the shelf tools cannot
   reach" is.

3. **Pick the surface, and say why.** If they must build, choose the thinnest wrapper around
   the capability they already have:
   - **A CLI the agent shells out to** - usually the simplest wrap. If the capability is
     already a script or a command, you are mostly done; you just give it clean flags and a
     clean stdout.
   - **An MCP server** - worth it when the tool is reused across many agents or needs a richer
     typed interface, but it is more to build and maintain.
   - **A plain function** - when the agent is your own code calling the model, not a shell.

   Default to the CLI unless they can name a concrete reason for the heavier option.

4. **Pin the one job.** Write a single sentence: "This tool <verb> <noun> and returns
   <thing>." If they need the word "and" twice, it is probably two tools. Split it.

5. **Record and hand off.** Capture the chosen surface and the one-job sentence, then move to
   `TOOL-INTERFACE.md` to design the actual contract.

---

## Pushback to give

Be concrete and a little annoying about scope:

- "You listed three things this tool does. Which one does the agent actually *call*? Are the
  other two really separate tools, or steps the agent can sequence itself?"
- "Does web fetch / shell / retrieval already do this? What specifically can't the shelf
  reach?"
- "Is this one tool or three? Say the job in one sentence with no 'and'."
- "What is the smallest surface that does the job - a CLI flag, or a whole MCP server?"
- "Who else runs this? If it is just this one agent, a CLI is probably enough."

---

## Both team types

- **Search-and-extraction:** "We have a patent database." Do you need a custom search tool at
  all, or does web fetch plus a small `patent-query` CLI over the DB suffice? Is "search" and
  "fetch full text" one tool or two? (Almost always two: a cheap search that returns ids, and
  a fetch-by-id - see phase 2.)
- **Experiment-guardian:** "We have an instrument driver." Wrap it as a read-only
  `get-reading` CLI, or as a full control tool that can also *change* the setpoint? Push them
  to start read-only: the guardian that can only observe cannot break the experiment, and you
  can add control later behind least privilege (phase 3).

---

## What this phase produces

A one-line record the team carries into phase 2:

```text
Tool name (working):   <name>
Surface:               CLI | MCP | function   (and the reason)
One job:               This tool <verb> <noun> and returns <thing>.
Off-the-shelf checked: <what you ruled out and why>
```
