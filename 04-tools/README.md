# Tool design - read these in order

These prompts are a **sequence, not a menu.** For one tool you walk all three phases in order,
each handing its decisions to the next:

1. **`TOOL-SCOPE.md`** - do you even need to build it? Build-vs-buy, pick the surface (CLI / MCP /
   function), pin the tool's one job.
2. **`TOOL-INTERFACE.md`** - design the agent-facing contract: name, description, parameters, and
   the terse return shape (the schema is the prompt).
3. **`TOOL-HARDEN.md`** - make it safe to run in a loop (determinism, good errors, idempotency,
   least privilege), then wrap it and smoke-test it.

Do not skip ahead: you cannot design a sensible interface before you have scoped the tool, and you
cannot harden an interface you have not designed.

`TOOL-SUPERVISE-JOBS.md` is **separate** - reach for it only when you are running several
long-running jobs in parallel that return at different times and need babysitting. It is about
the supervising agent, not about building a single tool.
