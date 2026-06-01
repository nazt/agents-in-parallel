# 10. Orchestration Shapes: Pipeline vs Barrier

*Two patterns govern every multi-stage parallel workflow. Choosing the wrong one doubles your wall-clock time for free.*

---

Most orchestration mistakes are not logic errors. The code is correct, the agents do the right work, the outputs compose cleanly — and the job still takes three times as long as it should. The culprit is almost always an unnecessary barrier.

## The Default: Pipeline

In a pipeline, each item moves through stages independently. Stage boundaries are not synchronization points; they are hand-off queues. Item A enters stage 2 the moment stage 1 finishes for item A, regardless of where items B, C, and D are.

The consequence for wall-clock time is significant. Suppose you have five items and three stages. Each stage costs roughly one second per item. In a pipeline, the last item exits stage 3 at approximately five seconds — one second of startup skew plus the slowest item's full chain. The stages overlap in calendar time. In a barrier version of the same job, you wait for all five items to clear stage 1 before any item starts stage 2, and again at stage 2 before stage 3. Wall-clock approaches three times the pipeline figure.

The formula is worth internalizing: **pipeline wall-clock equals the slowest single item's full chain, not the sum of the slowest-per-stage.** A barrier turns that sum back on.

This is why pipeline is the default. If you have no strong reason to synchronize, do not synchronize.

## When a Barrier Is Justified

A barrier earns its latency cost in exactly three situations.

**Cross-item deduplication before expensive downstream work.** If stage 2 will invoke a costly operation — a long model call, a database write, a network fetch — and stage 1 can produce duplicates, collecting all of stage 1's output first and deduplicating the set before launching stage 2 is correct. You are paying barrier latency once to avoid paying downstream cost N times. The math only works if the downstream cost per duplicate exceeds the barrier latency; confirm that before adding the synchronization.

**Early exit on empty set.** If the count of stage 1 results is zero, there is no point starting stage 2 at all. You cannot know the count is zero until all of stage 1 has reported. A barrier here is not overhead — it is the gate condition.

**Stage N must compare each item against the others.** Some operations are inherently cross-item: ranking, clustering, consensus voting across agent outputs, finding the global maximum. You cannot rank item A until you have seen item B through item N. If stage 2 is "select the three highest-confidence results from the full set," it requires a barrier. There is no pipeline-compatible version of that operation.

Everything else is not a barrier condition.

## What Is Not a Barrier Condition

Two false justifications appear repeatedly.

The first is "I need to flatten or filter the output before the next stage." Flattening a list-of-lists or filtering on a predicate is a map operation. It belongs inside a pipeline stage, not at a synchronization point. Each item's stage-1 output can be flattened and filtered as it arrives, and the result forwarded immediately into stage 2. Adding a barrier here serializes the pipeline for no reason.

The second is "the stages feel conceptually separate." Conceptual separation is about code organization, not execution order. A pipeline stage is already a discrete unit of work. Wrapping it in a barrier because it "feels like a different phase" imposes latency without buying correctness.

The test is strict: does stage N require data from *all other items* in stage N−1? If yes, barrier. If no, pipeline.

## The Latency Cost Is Real

Consider a team of coding agents doing a two-stage job: first, each agent probes a set of endpoints to collect health data; second, each agent synthesizes a report from the collected data. Suppose the five probes take 4 s, 4 s, 4 s, 4 s, and 12 s respectively. Stage 2 takes roughly 3 s per agent regardless.

In a pipeline, the four fast probes hand off at 4 s and their synthesis runs in parallel with the slow probe's remaining 8 s. Total wall-clock: 12 s (slow probe) + 3 s (its synthesis) = 15 s.

With a barrier between stage 1 and stage 2, all synthesis is blocked until the slow probe finishes at 12 s. Then all five syntheses run in parallel, finishing at 15 s. Same number — but only because synthesis is uniform. If synthesis time also varied, the fast agents would be idle waiting for the barrier, then idle again waiting for the slowest synthesizer. The losses compound.

The general principle: if the slowest stage-1 item takes *k* times the fastest, a barrier wastes *(k−1)/k* of the fast agents' available time. At k=3, you waste two-thirds. That is not a rounding error; it is the dominant cost.

## Composition

Pipelines and barriers compose. A multi-stage workflow might be:

```
pipeline → barrier → pipeline → pipeline
```

The barrier in the middle is where deduplication or a count check occurs. Everything else flows continuously. The shape of the workflow should match the data dependencies, not the programmer's intuition about "phases."

When designing a new multi-stage job, start by sketching data dependencies between stages: does stage N+1 need one item's stage-N output, or all items' stage-N output? The answer directly dictates the shape. One item's output → pipeline. All items' output → barrier.

---

**The rule:** default to pipeline; add a barrier only when a stage genuinely requires cross-item context from the complete previous result set, and verify that the downstream savings exceed the synchronization tax before you pay it.