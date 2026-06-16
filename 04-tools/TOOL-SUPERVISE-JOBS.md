# Babysit long-running, staggered parallel jobs

**This file is a prompt for the agent.** A team hands you this when they are running 3 to 10
jobs in parallel that finish at *different* times - long extractions over many documents, or
several live experiment runs watched at once - and they need a supervising agent to ride herd
on them. Your job is to brainstorm the design, push back hard on the parts that will tangle or
hang, and help them build the per-job dirs, the status protocol, and the stop rule. The
teaching point: the supervisor is a **small-context reaper**, not a worker. It launches jobs,
holds *handles and status*, and never reads the raw job output into its own context. The
moment the supervisor starts ingesting what the jobs produce, it stops being a supervisor and
becomes a bottleneck that rots and crashes.

This sits at the seam between the **fan-out** pattern (`../02-patterns/PATTERN-FANOUT.md`) and
tool design. The "tool" here is the supervisor plus the status surface the jobs report through.
It is the workhorse session for experiment-guardian teams, and the parallel-search teams use it
too.

---

## How to run this session

1. **Establish the frame first.** Confirm with the team: the supervisor holds handles and
   one-line statuses, never the outputs. If they are planning to "collect all the results and
   then decide," stop them - that is the bottleneck. Each job's output stays in the job's own
   dir; the supervisor reads a tiny status file.
2. **Work the two halves below:** keep the jobs unentangled, then design the stopping rule.
   Do the stop rule *before* any code spawns - it is the most common thing teams skip and the
   most expensive to bolt on later.
3. **Build the three pieces:** the per-job dir + status-file convention, a poll loop that reads
   statuses (not outputs), and an explicit stop-rule check inside that loop.

---

## Half 1: keep the jobs unentangled (isolation)

Every job gets its own everything. No shared mutable state. If two jobs can write the same
file or fight over the same resource, you do not have ten jobs - you have one tangled job
wearing ten hats, and a failure in any one corrupts the rest.

- **Own workspace:** each job runs in its own dir - a stage-dir or a per-job sub-dir inside the
  scaffold run (see `../05-scaffold/SCAFFOLD.md`). It reads its inputs and writes its outputs
  only there.
- **Own id:** a stable job id used for its dir name, its status file, and its log. Results are
  attributed by id, never by "the order they came back" - staggered returns make arrival
  order meaningless.
- **Own status file + log:** the job's only channel to the supervisor.
- **Failure isolation:** one job hanging, dying, or going rogue must not block, corrupt, or get
  mis-attributed to the others. Test this on purpose - kill one job and confirm the rest finish
  clean.

**Status protocol.** Each job writes a tiny status file the supervisor *polls* (it does not
stream output up): one of `pending | running | done | failed | needs-attention`, plus a
one-line summary and a pointer to where the full result lives. The supervisor reads ten short
status files, never ten large outputs.

- **Search-extraction:** fire 10 patent/paper queries; each writes to
  `.../02-search/<query-id>/status` and drops its hits in `.../02-search/<query-id>/result.json`.
  The supervisor sees ten one-liners, not ten result sets.
- **Experiment-guardian:** watch several live runs; each run reports to its own
  `.../runs/<run-id>/status` with `running: temp 71C, nominal` or
  `needs-attention: pressure climbing`. One stuck run never blocks the view of the others.

> Pushback: "Are you holding job *output* in the supervisor, or just handles and statuses? Do
> any two jobs touch the same file or resource - name it. What does a *hung* job look like to
> you versus a merely *slow* one - if you cannot tell them apart from the status file, you
> cannot stop the hung one."

## Half 2: design the early stopping rule

Decide how this batch ends **before you spawn anything.** Staggered jobs mean you are always
waiting on *someone*; without an explicit rule you wait on the slowest, forever if it hangs.

**Per-job stops:**

- a timeout - past it, the job is declared `failed` and reaped;
- a cost ceiling - tokens or wall-clock budget per job;
- a self-report - a job that detects it is going nowhere sets `failed` with a reason rather
  than spinning.

**Whole-batch stops - pick one on purpose:**

- **wait-for-all** - the trap. One straggler stalls the whole batch. Only choose this if you
  genuinely need every result *and* every job has a hard timeout.
- **first-success-wins** - take the first good result, cancel the rest.
- **quorum / N-good-out-of-M** - stop once you have enough good answers; cancel stragglers.
- **deadline** - stop at time T with whatever is done; mark the rest incomplete.
- **guardrail trip** - one job detects the bad thing and you halt the rest immediately. This
  is the experiment-guardian's core case.

Examples:

- **Search-extraction:** quorum - "I need 5 solid hits; fire 10 queries, stop and cancel the
  rest once 5 land." The two slow queries never hold up the pipeline.
- **Experiment-guardian:** guardrail trip - "if any run trips a danger threshold, halt the
  batch and alert; otherwise run to the deadline." Plus per-run timeouts so a frozen rig
  reads as `failed`, not `running forever`.

> Pushback: "Write your stop condition down before you spawn. You said 'wait for all' - what
> is your plan when job 7 never returns? What cancels the stragglers once you have enough, and
> does cancelling them leave their dirs in a clean state?"

---

## Build the three pieces

1. **Per-job dir + status-file convention** - one dir per job under the scaffold run, each with
   a `status` file and a `log`. Agree the exact status vocabulary above.
2. **Poll loop** - the supervisor reads the status files on an interval, builds a small table
   of `id -> state -> one-liner`, and reads *nothing else*. It never opens a `result.json`
   except to hand its path onward.
3. **Stop-rule check** - inside the poll loop, evaluate the chosen whole-batch rule each tick;
   when it fires, cancel outstanding jobs and reap. Handle per-job timeouts in the same tick.

The basis for spawning the jobs is the fan-out pattern (`../02-patterns/PATTERN-FANOUT.md`);
the dirs come from `../05-scaffold/SCAFFOLD.md`. This file adds the part fan-out leaves out:
the jobs return at different times, and someone has to decide when to stop waiting.
