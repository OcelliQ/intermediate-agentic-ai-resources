# Workshop resources: starter prompts

These are **starter prompts** for the Intermediate Agentic AI workshop. Each file is a prompt
you hand to *your agent* (Claude Code) at the start of a build session - not a document you read
on your own. You open one, paste it (or point your agent at it), and the agent runs a working
session with you: it helps you brainstorm, pushes back on weak assumptions, and helps you build
the concrete artifact.

The goal is not to hand you answers. It is to give your agent a script for asking you the right
questions, in the right order, so that what you build is reliable, observable, and testable.

## Two kinds of teams

- **Search-and-extraction** - bots that search and extract structured information from patents,
  papers, and documents.
- **Experiment-guardian** - bots that watch over running scientific experiments and alert or act.

Every prompt carries examples for both. Read the one that matches you; the other is usually a
useful contrast.

## Suggested order of use

1. **Pick your properties** - figure out which of the 8 problems your task actually has.
2. **Pick your patterns** - choose the workflow shapes that solve those problems.
3. **Design the seams** - define handoffs between stages, and design any tools you need.
4. **Scaffold** - lay down the directory structure your runs live in.

You will loop back through these as you build. That is expected. Start simple, get one stage
working, then grow.

## The files

### `01-properties/`

- **`PICK-PROPERTIES.md`** - work out which of the 8 properties (context rot, credulity,
  abstraction, specialization, observability, least privilege, testability, cost & latency)
  your task needs you to engineer for, and which you can ignore.

### `02-patterns/`

- **`PATTERN-PIPELINE.md`** - a fixed sequence of stages with deterministic gates between them.
- **`PATTERN-FANOUT.md`** - many children read wide in parallel; the parent stays small and
  synthesizes.
- **`PATTERN-ROUTING.md`** - classify first, then dispatch down the right path with the right
  model and tools.
- **`PATTERN-ACTOR-CRITIC.md`** - generate, then have a fresh-context critic and a judge weigh
  the result.

### `03-handoff/`

- **`DESIGN-HANDOFF.md`** - define what one agent passes to the next, for both free text and
  structured data: payload, status, gaps, references, and the contract that gets enforced.

### `04-tools/`

- **`TOOL-SCOPE.md`** - decide build-vs-buy, pick the tool's surface (CLI / MCP / function), and
  pin its one job before you write anything.
- **`TOOL-INTERFACE.md`** - design the agent-facing contract: the schema is the prompt, and the
  return shape must be token-efficient.
- **`TOOL-HARDEN.md`** - make the tool safe to run in a loop (good errors, determinism,
  idempotency, least privilege), then wrap it and smoke-test it.
- **`TOOL-SUPERVISE-JOBS.md`** - design an agent that babysits 3-10 long-running jobs that
  return at different times: keep them unentangled and decide when to stop early.

### `05-scaffold/`

- **`SCAFFOLD.md`** - set up the `pipeline-<name>/run-<datetime>/<stage-number>-<stage-name>/`
  directory structure. This is where provenance, ownership, and idempotency live, so start here
  early - the handoff and tool prompts reference it.

## Also here: `demo/`

The `demo/` folder holds the live-demo prompts run during the talk itself (for example, the
fan-out and actor-critic demos). Those are run by the presenter on stage; the starter prompts
above are for you to run during the build session.
