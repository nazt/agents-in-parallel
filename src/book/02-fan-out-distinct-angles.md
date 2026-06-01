# 2. Fan Out With Distinct Angles

*Parallelism is only useful when each agent is asking a different question.*

---

The naive version of multi-agent parallelism is five workers all running the same search. You get five answers. They agree or they disagree. Either way you have learned approximately nothing you could not have learned from one. The agents were parallel in execution; they were not parallel in coverage.

The useful version is N agents, each assigned a non-overlapping slice of the problem space. When they finish, the union of their outputs is larger than any single agent could have produced. That is the whole mechanism. Diversity of method is not a nice-to-have — it is the point.

## What "Distinct Angles" Actually Means

An angle is a combination of: the data source you query, the tool you invoke, and the question you are trying to answer. Two agents have distinct angles when changing one of those three things would materially change the answer.

"Search documentation for X" and "search documentation for Y" are not distinct angles. They use the same source and the same tool; they just parameterize differently. A single agent could do both sequentially in seconds.

"Search documentation for X" and "read the live process table for X" are distinct angles. One looks at declared intent; the other looks at observed behavior. They can and do disagree, and that disagreement is information.

The practical test: if the two agents could swap instructions mid-run and produce equivalent output, they are not distinct.

## Angle Taxonomies by Domain

Angles are domain-specific. Here are three common shapes.

**Research.** Partition by source type: primary sources (original papers, RFCs, specs), secondary analysis (synthesized commentary, blog posts), structured databases (version tables, vulnerability registries), and informal practitioner writing (forums, mailing lists, issue trackers). A research lead assigns one agent per tier. The outputs differ by epistemic status, not just content, which is why the compilation step is non-trivial.

**Infrastructure diagnosis.** Partition by layer: network reachability, service-level response (HTTP, RPC), configuration on disk (registry files, manifests), live process state (what is actually running), and source-of-truth reads (what the codebase says should be running). Each layer has a distinct failure mode. A network failure looks like silence from all other layers; a misconfigured registry looks like a healthy network but a broken service. You cannot infer layer N from layer N−1.

**Build or refactor.** Partition by component. One agent owns the persistence layer, one owns the API surface, one owns the front-end state model. They operate on different files, commit to different branches or worktrees, and block each other only at integration. The lead's job is sequencing the merge, not supervising the work.

## Proof: A Five-Layer Fleet Map

A five-agent health probe ran against a fleet of machines connected by a mesh VPN, each running a collection of services registered in a shared registry. The question was: what is actually running, and does it match what should be running?

One agent pinged the network layer — pure reachability, no application protocol. It returned which hosts were visible at the transport level.

A second agent probed the service endpoints on each visible host via HTTP — health checks, version strings, response latency. It returned which declared services were responding.

A third agent read the registry on disk — the configuration files that described what each host was supposed to run. It returned the declared state.

A fourth agent queried the live process roster on each host via shell. It returned what was actually executing, including processes that had no registry entry.

A fifth agent read the relevant section of the transport source — the part of the codebase that defines how services register and deregister themselves. It returned the authoritative rules for what a valid registration looks like.

The lead compiled the five outputs into a single table: expected vs. declared vs. live. The result was a complete fleet map that no single layer could have produced:

- Two hosts were network-reachable but had no registry entries (discovered by comparing agent one and agent three).
- One registered service was returning HTTP 200 but the process roster showed the wrong binary version (discovered by comparing agents two and four).
- The source agent revealed that a registration field used by several hosts was deprecated and silently ignored by the runtime — meaning the registry entries were valid-looking but semantically dead.

None of these findings were reachable by repeating any single layer's query. The value came entirely from the joins across distinct outputs.

## The Failure Mode: Angle Collapse

Angle collapse happens when instructions drift toward similarity at authoring time. A lead writes five tasks, each phrased slightly differently, but each bottoming out in the same operation: semantic search against the same index. The agents fan out, run in parallel, and return five variants of the same answer. The lead spends time synthesizing output that contains no new information.

The fix is to write the task list as an explicit inventory of sources and tools, not an inventory of questions. Before spawning, ask: which data source does each agent own? If two agents share a data source and a tool, merge them into one.

A second failure mode is premature sharing. Agents that can observe each other's partial results mid-run will converge. The lead sees a tidy consensus that hides the variance that would have revealed the real finding. Keep agents blind to each other until the compilation step.

## The Compiler Role

The lead agent is not a voter. It is not averaging the outputs or picking the plurality answer. It is a compiler: it takes outputs with known provenance (this came from the network layer; this came from the source) and constructs a representation that is only possible because the inputs were generated independently from different angles.

Compilation requires knowing what each agent was looking at. The task assignment must carry that provenance, not just the result. An agent that returns "service X is healthy" is less useful than one that returns "service X returned HTTP 200 from the health endpoint at layer 2; process roster was not checked." The qualifier tells the compiler where the finding fits in the join.

---

**The rule:** assign agents by data source and tool, not by topic — if two agents could swap tasks and return equivalent output, they are not distinct enough to parallelize.
