# Scaffold your pipeline's directories

**This file is a prompt for the agent.** When a team hands you this file, help them lay down the
directory structure their pipeline runs in - *before* they write stages, handoffs, or tools.
The teaching point: a good directory layout is not bookkeeping. It is how this workshop *carries*
provenance, ownership, and idempotency for free, so that handoffs between agents do not have to
re-encode any of it in prose. Get the dirs right and three of the eight properties come for free.

## The canonical layout

```text
pipeline-<name>/
  run-<datetime>/
    <stage-number>-<stage-name>/
      <files>
```

For example:

```text
pipeline-patent-search/
  run-2026-06-16T14-30-00/
    01-search/        queries.json, raw-hits.jsonl, status.json
    02-extract/       records.jsonl, status.json
    03-validate/      validated.jsonl, rejects.jsonl, status.json
    04-report/        report.md, status.json
  run-2026-06-16T16-05-12/
    ...
```

Each level earns its place. Make the team say out loud what each one means for *their* pipeline:

- **`pipeline-<name>/`** - one named workflow. Everything that belongs to "the patent searcher"
  or "the reactor watcher" lives under one of these. The name is stable across runs.
- **`run-<datetime>/`** - one execution, and it is **immutable**. This is where idempotency and
  provenance live: a re-run is a *new* run dir, never an overwrite of an old one. If you ran it
  last Tuesday and again today, you have two run dirs and can diff them. Nothing inside a finished
  run dir should ever be rewritten.
- **`<stage-number>-<stage-name>/`** - one stage's (one agent's) workspace. The number forces an
  order you can read at a glance; the name says what the stage does. **Ownership rule: a stage
  writes only its own dir.** It reads from earlier stages, it writes to its own. That single rule
  is what keeps stages from clobbering each other.
- **`<files>`** - the actual artifacts: handoff envelopes passed to the next stage, source
  material pulled in, logs, and a `status.json` the stage writes when it finishes.

Because the run dir is immutable and each stage owns its own dir, you do not have to write
"produced by stage 2 of run X at time T" into every handoff - the path already says it. See
`../03-handoff/DESIGN-HANDOFF.md`; handoffs should reference files by their path in this tree
(for example `../01-search/raw-hits.jsonl`) rather than copying them.

## How to run this session

1. **Name the pipeline.** One short, stable, hyphenated name. Push back if it describes a single
   run ("tuesday-run") rather than the workflow ("patent-search", "reactor-watch").
2. **Enumerate the stages.** Walk the team from input to output and name each stage as a verb:
   what does it *do*? Number them `01`, `02`, ... Keep the count small to start - you can split a
   stage later.
   - Search-and-extraction, for example: `01-search`, `02-extract`, `03-validate`, `04-report`.
   - Experiment-guardian, for example: `01-poll-sensors`, `02-check-thresholds`, `03-alert`.
3. **Agree the datetime format.** Use a sortable, filename-safe form so runs list in time order
   and never collide: `run-2026-06-16T14-30-00` (no colons that fight the filesystem; lexical
   sort = chronological sort). Decide local vs UTC now and write it down.
4. **Create the dirs**, then **offer to write a tiny helper** that mints a fresh run. This is
   exactly the kind of small automation you should build for them. For example:

   ```sh
   # new-run.sh <pipeline-name> <stage-name>...
   pipeline="pipeline-$1"; shift
   run="$pipeline/run-$(date +%Y-%m-%dT%H-%M-%S)"
   i=1
   for stage in "$@"; do
     mkdir -p "$run/$(printf '%02d' "$i")-$stage"
     i=$((i+1))
   done
   echo "$run"
   ```

   so `./new-run.sh patent-search search extract validate report` prints the new run dir and
   makes its stage subdirs. Adapt it to their names.

## Pushback to give them

- **What is immutable vs. rewritten across a re-run?** If their instinct is to overwrite
  yesterday's output in place, stop them: a re-run is a new run dir. Otherwise they cannot tell
  what changed, and a half-finished re-run can corrupt a good result.
- **If two stages want to write the same file, who owns it?** Exactly one stage owns each file.
  If two need it, that is a handoff (the owner writes, the other reads), not shared mutable state.
- **How do you find last Tuesday's run?** If the answer is "scroll and guess", the datetime
  format is wrong. Sortable names make this `ls` + read.
- **Where do large sources live?** Inside the run dir, so handoffs can point at them by relative
  path and line range instead of pasting them. Keep the inputs next to the run that used them.
