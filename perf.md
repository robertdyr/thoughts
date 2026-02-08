# Debugging Slow Systems: Failure Modes, Observability, and Why There Is No Single Root Cause

## Motivation

Debugging performance problems often feels unsatisfying and inconsistent:

* Two engineers analyze the same slow system and propose different fixes.
* Metrics clearly show *something* is wrong, but they don’t directly tell you *what to change*.
* Sometimes adding an index is the “right” fix; other times adding CPU solves the problem just as well.
* Experience seems to matter more than logic — but it’s unclear why.

This document captures a mental model that explains **why debugging behaves this way**, what can be determined **deterministically**, and where **knowledge, observability, and judgment inevitably enter**.

The core insight is simple but non-obvious:

> Debugging is not about finding *the* root cause.
> It is about navigating observable behavior, failure modes, and structural explanations under irreducible limits.

---

## 1. Observability Comes First

We can only reason about what we can measure.

Typical observable metrics include:

* CPU utilization
* User vs kernel CPU time
* IO wait
* Lock wait
* Row scan counts
* Query execution statistics

From these measurements, we can classify **what the system is doing**, but not **why it should or should not be doing it**.

This leads to the first hard limit:

> You cannot reason about attributes you do not know exist, and you cannot use metrics the system does not expose.

### Knowledge and Instrumentation Both Matter

Failure-mode refinement depends on **two independent prerequisites**:

1. **Instrumentation**
   The system must expose the metric (e.g. user vs kernel CPU time).

2. **Knowledge**
   The engineer must know that:

   * the distinction exists,
   * where to find it,
   * and how to interpret it.

If either is missing, the refinement is unavailable.

Two engineers — or two Linux distributions — may therefore observe **different effective systems**, even if the underlying behavior is identical.

This makes refinement inherently **knowledge- and tooling-dependent**, not purely logical.

---

## 2. Failure Modes Are Behavioral Equivalence Classes

A **failure mode** is defined purely by **observable runtime behavior**.

Examples:

* `B_cpu`
  → CPU is the binding resource

* `B_cpu,user`
  → CPU time is mostly user-space

* `B_cpu,user,row-scan`
  → CPU time is dominated by scanning rows

Each refinement is only possible if:

* the metric exists,
* it is exposed,
* and the engineer knows to look for it.

Importantly:

> Failure modes describe *what kind of work dominates*, not *whether that work is justified*.

They are **behavioral equivalence classes**:
different systems with different causes can belong to the same failure mode if they look identical at runtime.

---

## 3. One Failure Mode → Many Structural Explanations

A single failure mode maps to a **set of possible structural explanations**, not a single cause.

Example:

```
Failure mode: B_cpu,user,row-scan
```

Possible explanations include:

* Missing index
* Wrong index
* Query requires full scan
* Too much concurrency
* Insufficient CPU capacity
* Inefficient query shape
* Poor data layout

At this stage:

* nothing uniquely determines the “root cause”
* and no amount of behavior alone can

This is not a failure of analysis — it is inherent.

---

## 4. Warranted vs Unwarranted Work Is Not Observable

Consider two cases:

### Case A — Warranted (W)

* Query: `SELECT * FROM large_table`
* No `WHERE` clause
* Full table scan is semantically required

### Case B — Unwarranted (U)

* Query should filter a small subset
* Missing or wrong index causes full scan

**Observed behavior in both cases:**

* High CPU
* High row scan count

Therefore:

> From observation alone, you cannot distinguish warranted from unwarranted work.

This distinction depends on **workload semantics and intent**, which are **not encoded in runtime metrics**.

This is a fundamental limit of observability.

---

## 5. Workload Knowledge Does Not Refine Failure Modes

Workload knowledge (query semantics, business intent) does **not** refine the failure mode.

Both Case A and Case B remain:

```
B_cpu,user,row-scan
```

What workload knowledge does instead:

> It **filters the set of structural explanations**.

Example:

* Knowing the query is `SELECT *` without a `WHERE` clause eliminates:

  * missing index
  * bad selectivity
* Leaving:

  * CPU sizing
  * batching
  * scheduling
  * architectural redesign

Thus:

* **Behavior defines failure modes**
* **Workload constrains explanations**

These are different roles and must not be conflated.

---

## 6. Refining Failure Modes vs Pruning Explanations

Failure-mode refinement is useful **only while it meaningfully reduces the solution space**.

Examples where refinement is valuable:

* CPU → user vs kernel (very different fixes)
* IO → sequential vs random
* Locks → one blocker vs many

Examples where refinement stops helping:

* Warranted vs unwarranted row scans
  (cannot be resolved behaviorally)

When refinement no longer reduces solution classes, further metric digging has diminishing returns.

At that point, the correct move is to:

> Stop refining behavior and switch to workload- and structure-based reasoning.

---

## 7. There Is No Single “Correct” Solution

Once explanations are pruned, multiple valid interventions usually remain.

For `B_cpu,user,row-scan (warranted)`:

* Add CPU
* Reduce concurrency
* Batch or schedule queries
* Precompute or summarize
* Redesign architecture

For `B_cpu,user,row-scan (unwarranted)`:

* Add or fix index
* Rewrite query
* Change access pattern

All of these can relieve the binding resource.

Which one is “best” depends on:

* cost
* risk
* operational complexity
* organizational constraints
* long-term goals

Therefore:

> Choosing a solution is a **value judgment**, not a logical deduction.

This explains why different engineers may choose different fixes — and both can be correct.

---

## 8. What This Model Explains

This framework explains why:

* Metrics alone never “tell you the answer”
* Experience matters (richer attribute vocabulary)
* Observability tooling changes debugging outcomes
* “Root cause analysis” often oversimplifies reality
* Debugging feels partially deterministic and partially judgment-based

---

## Final Takeaways

* Failure modes are behavioral equivalence classes
* Refinement is limited by instrumentation *and* knowledge
* Behavior cannot encode intent
* Workload semantics prune explanations, not behaviors
* Multiple solutions are always possible
* Debugging is deterministic up to the failure mode
* Engineering judgment begins at the solution

Or in one sentence:

> **Debugging is the process of mapping observable behavior to a failure mode, then using workload and structural reasoning to choose among multiple valid interventions—under irreducible limits of observability and knowledge.**

