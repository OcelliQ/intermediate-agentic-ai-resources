# Pattern: Routing / triage

**This file is a prompt for the agent.** A team is about to design a routing step and has
handed you this file to run the session. Routing means: a cheap classifier looks at each input,
does ONE thing - decides which class it is - and dispatches it down a single specialized path
with the right model, prompt, and tools. Easy inputs go to a small, cheap path; only the hard
inputs pay for the expensive one. Your job is to help the team enumerate their real input
classes, define the classifier's narrow job, design each downstream path, and - critically -
make sure the router itself holds no power it does not need. Push back whenever the router
starts doing the work it should be delegating, or whenever the team forgets the inputs that do
not fit any class.

---

## Why routing (which of the 8 properties it buys)

- **Cost & latency (8)** - the classifier is small and fast; the cheap path handles the common
  easy cases; only the rare hard case pays for the big model.
- **Specialization (4)** - each downstream path is a focused agent with one job, a tight
  prompt, and exactly the tools that path needs.
- **Least privilege (6)** - the router itself has almost no power. It classifies and dispatches;
  it does not hold the downstream tools. A wrong classification sends work to the wrong
  specialist - it does not hand dangerous tools to the wrong place.

---

## How to run this session

1. **Enumerate the real input classes.** Have the team list the kinds of input they expect.
   Then press for the messy ones: the ambiguous input, the malformed input, the input that
   fits two classes, the input that fits none. Every real router needs a default/fallback path.
   - Search-extraction: route by document type - patent vs. journal paper vs. press release -
     to the extractor built for that format. Or route by query intent - a quick lookup vs. a
     deep multi-field extraction - so trivial questions do not invoke the heavy pipeline.
   - Experiment-guardian: route by signal severity - a normal reading goes straight to the log;
     a suspected anomaly goes to a deep-check agent; a danger threshold goes to an immediate-
     alert path. Or route by instrument type to the watcher specialized for that feed.

2. **Define the routing key and the classifier's narrow job.** The classifier outputs one
   thing: the class label (plus maybe a confidence). It does not extract, summarize, or act -
   it only labels and dispatches. Write the exact set of output labels, including the fallback
   label. Keep the classifier prompt tiny and the model cheap.
   - **Often the router IS the orchestrator.** In many designs there is no separate classifier
     agent at all: the orchestrator - the agent running the whole show - makes the routing
     decision itself, inline. That is common and perfectly fine, but it has a cost - the decision
     now shares the orchestrator's context. So keep it SHORT and cheap: a label picked from a
     fixed list, not a sprawling analysis that loads up the orchestrator's window. If choosing the
     route genuinely needs heavy reading or deep reasoning, push it back out into its own small
     classifier agent so the orchestrator's context stays clean.

3. **Design each downstream path.** For every class, the team should name the path's model, its
   prompt, and its tool set. This is where specialization and least privilege are spent well:
   the FAQ-style path gets a small model and read-only tools; the alert path gets exactly the
   alerting tool and nothing else. Make them write one path per class plus the fallback.

4. **Decide what happens on a wrong or low-confidence classification.** Routing's main failure
   mode is misclassification. Options: a default/fallback path, a confidence threshold that
   escalates ambiguous inputs to a human or to the safe path, or a cheap downstream check that
   bounces mis-sent work back. Pick one deliberately.

---

## Pushback to keep applying

- "What are your real input classes - and what happens to the input that fits none of them, or
  two of them? Show me the fallback path."
- "Is the router doing work it should delegate? If your classifier is reading the whole
  document to decide, it is no longer cheap and no longer just a router."
- "When the classifier is wrong, what is the blast radius? An anomaly mislabeled as 'normal'
  gets silently logged and ignored - is that acceptable? Bias your default toward the safe
  path."
- "Who actually makes the routing call - a separate classifier, or the orchestrator inline? If
  it is inline, is the decision short enough that it does not bloat the orchestrator's context?
  A label from a fixed list is fine; a paragraph of analysis per route is not."
- "Does the router hold any downstream tools? It should not. It classifies and hands off;
  the power lives in the specialized paths."

---

## What good looks like

The classifier is a small, fast, one-job agent with a fixed list of output labels and an
honest fallback. Each class has its own path with the cheapest model that does the job and the
narrowest tool set. The expensive path runs only on the inputs that truly need it. And a
misclassification is contained - it sends work to the wrong specialist or to a safe default,
not to disaster.
