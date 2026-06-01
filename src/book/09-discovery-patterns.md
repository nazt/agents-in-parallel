# 9. Discovery Patterns: Loop-Until-Dry and Multi-Modal Sweep

*When you don't know how big the answer is, count down to silence — not up to a quota.*

---

The most dangerous assumption in agent orchestration is that you know the size of what you're searching for. Count the bugs in a codebase. Find every stale reference in a distributed config. Locate all the unhealthy nodes in a service mesh. In each case, you are looking for something whose population is unknown, and an arbitrary stopping criterion — "run five finders and collect results" — will miss the tail systematically.

Two patterns address this. Loop-Until-Dry handles depth: keep searching until the search demonstrably stops yielding. Multi-Modal Sweep handles coverage: search the same space several ways simultaneously, because no single approach finds everything.

---

## Loop-Until-Dry

The structure is simple. Spawn a round of finder agents. Dedup their output against a global SEEN set. If the new-findings count is above zero, start another round. If K consecutive rounds surface nothing new, stop.

The dedup rule is load-bearing. *You must dedup against everything ever surfaced, not just everything that survived downstream review.*

Here is why that matters. Suppose a finder surfaces a candidate. A judge agent reviews it and rejects it — not a real bug, false positive. On the next round, another finder surfaces the same candidate. If SEEN only tracks accepted findings, this candidate is new to SEEN and gets routed to the judge again. The judge rejects it again. Round after round the same false positives circulate, the new-findings count never reaches zero, and the loop never converges.

The fix: add to SEEN at intake, before review. The judge still operates on the full set, but the loop's convergence signal is deduplication-before-review. Rejected items stay in SEEN with a `rejected` tag. They never appear in output. They do appear in the convergence check.

```
seen = {}           # keyed by canonical fingerprint
round_new = []

for each round:
    candidates = run_finder_wave()
    for c in candidates:
        key = fingerprint(c)
        if key not in seen:
            seen[key] = c
            round_new.append(c)

    if len(round_new) == 0:
        consecutive_empty += 1
    else:
        consecutive_empty = 0

    if consecutive_empty >= K:
        break
```

The fingerprint function matters too. For a bug report it might be `(file, line_range, error_class)`. For a service-mesh node it might be `(node_id, failure_mode)`. Sloppy fingerprinting — just the node ID, without the failure mode — merges distinct findings and makes the loop terminate too early.

---

## Budget Guard

An open-ended loop needs a hard ceiling. The convergence signal is the primary brake; the budget guard is the emergency brake.

Track two numbers: `rounds_elapsed` and `total_agent_calls`. Set ceilings on both before the loop starts. When either ceiling is hit, the loop stops and logs a `budget_exceeded` event — so you know the search was truncated, not complete. Silence-after-K-rounds means done. Budget hit means unknown.

This distinction matters for callers. A downstream system that consumes "all live unhealthy nodes" should treat a budget-hit result differently from a clean-convergence result — perhaps escalate, perhaps widen the budget, perhaps flag the report as partial.

---

## Multi-Modal Sweep

Loop-Until-Dry handles depth. Multi-Modal Sweep handles breadth. The premise: any single search strategy has blind spots, and the blind spots are often invisible from inside that strategy.

Consider finding every reference to a deprecated API in a large repository. Strategy A searches by symbol name. Strategy B searches by file-level import patterns. Strategy C searches by commit message for the migration flag. Strategy D searches by the test suite for test names that encode the old API name.

Run all four in parallel, with no inter-agent communication during the sweep. Each agent is deliberately blind to what the others find. They don't coordinate; they compete to find things the others miss. You collect results afterward and dedup into a merged set.

Why keep them blind during the sweep? Shared state during search creates anchoring bias. If finder B sees that finder A already found `src/auth/token.go`, B will unconsciously (or explicitly, if prompted carelessly) deprioritize similar files. The whole value of multi-modal sweep is that each strategy covers its own blind spots. Cross-pollinating during the sweep defeats this.

---

## A Concrete Case

A team ran a sweep to find every misconfigured endpoint in a service registry — roughly four hundred services, each with several config fields. They dispatched five parallel finder agents: one searched by container label, one by config file pattern, one by named entity (service name prefix), one by last-modified timestamp window, and one by cross-referencing the live-state file against the charter file.

Round one surfaced sixty-one candidates. Round two, after dedup against the full round-one set, surfaced nineteen new ones. Round three: four. Round four: zero. Round five: zero. Two consecutive empty rounds was the configured K. Loop stopped.

Without multi-modal sweep, the label-only search would have found forty-three of those eighty-four. The timestamp-window search found eleven that no other strategy touched — services that had been renamed, breaking the label match, but the rename timestamp sat in the known migration window.

Without the dedup-before-review rule, the loop would have churned on eight false positives that the judge kept rejecting. Observed behavior in an early version: still running at round twenty-two.

---

## Composition

The two patterns compose naturally. Run a multi-modal sweep as each round of a Loop-Until-Dry. Each wave dispatches N strategies in parallel; their merged output is dedupped into SEEN; the convergence check fires on the merged-and-dedupped delta, not per-strategy. Budget guards apply to the outer loop, not to individual strategies.

The structure looks like concentric loops: outer loop checks convergence, inner fan-out handles strategy diversity. Each layer has a single responsibility.

---

**The rule:** when the answer set has unknown size, measure by convergence — count down to silence across K empty rounds, dedup at intake (not after review), cover the search space from multiple independent angles, and let a hard budget guard distinguish "done" from "stopped early."