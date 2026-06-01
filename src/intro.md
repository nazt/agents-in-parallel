# Parallel Agents: Method, Contradiction, Reconcile

*A practitioner's field guide to orchestrating many AI agents correctly — at the same time.*

---

## Preface

The promise of running multiple agents in parallel is speed. That is the wrong reason to do it. Speed is a side-effect. The real value comes from three mechanisms that only work when agents are genuinely independent and methodologically distinct. This book is about those three mechanisms, why they work, and how to build infrastructure that makes them reliable.

**The first mechanism is method diversity.** A single agent makes a single set of methodological choices — which layer to probe, which path to follow, which abstraction to trust. Those choices are invisible when the agent produces a correct answer and catastrophic when it produces a confident wrong one. When you run five agents in parallel with five genuinely different angles — one probing at the network layer, one at the service layer, one reading the registry on disk, one inspecting the process tree, one reading the source — you don't just get five data points. You get five chances for two agents to directly contradict each other, and that contradiction is more informative than any individual answer. A single agent probing a broken hostname-resolution layer will conclude a node is unreachable. Two agents — one using the hostname, one using a raw IP address — will contradict each other and point directly at the broken layer. The contradiction is the diagnosis.

**The second mechanism is adversarial verification.** Parallel agents can be tasked not to produce findings but to attack them. Before any irreversible action — publishing, deploying, sending, committing a claim to memory — one agent's job is to find reasons the main agent's output is wrong. Not to help, not to refine, but to refute. This is not the same as asking an agent to review its own work; it is structurally different. An adversarial sub-agent that finds nothing wrong is genuine evidence. The same agent that produced the claim searching its own work for flaws is not. The asymmetry matters because irreversible actions have asymmetric costs.

**The third mechanism is idempotent declare-then-reconcile.** A team of agents is not an ephemeral arrangement of running processes. It is a specification: a charter that says who is on the team, an engine map that says how to spawn each member, and a live-state file that says what is running right now. The first two files are durable and version-controlled. The third is disposable — it is the output of reconciling the declaration against the current machine state. Every operation that instantiates a team should be a re-runnable move toward the declared configuration. If a pane is missing, the operation creates it. If the declared isolation is not true, the operation does not patch around it — it destroys and recreates. A declaration is a request; reconciliation is the proof that it was honored.

These three mechanisms are independent of any particular tool. They do not require a specific agent runtime, a specific model vendor, a specific orchestration framework, or a specific host. They require only that you actually implement them, rather than assuming they emerge from running multiple agents at the same time.

---

### Who This Book Is For

Engineers who are already running one agent and want to understand when and why to run several — not as a strategy for parallelism, but as a strategy for correctness. Practitioners who have found that agent outputs can look confident and be wrong, and want structural defenses against that, not just better prompts. Teams building orchestration layers who want durable patterns, not vendor-specific recipes.

---

### How to Read It

The first four chapters establish the core mechanisms and the failure modes they address. Read them in order. Chapters five through eight build the operational infrastructure: how to specify teams as files, how to gather results, how to verify the right properties, and how to isolate parallel writers. The final four chapters cover the meta-patterns: discovery strategies, orchestration topologies, synthesis discipline, and when to stop abstracting. Each chapter stands on its own for reference, but the arguments build on each other.

---

## Table of Contents

1. [Why Parallel Agents](book/01-why-parallel.md)
2. [Fan Out With Distinct Angles](book/02-fan-out-distinct-angles.md)
3. [Contradiction Is the Signal](book/03-contradiction-is-signal.md)
4. [Adversarial Verification](book/04-adversarial-verification.md)
5. [The Team Is a File](book/05-the-team-is-a-file.md)
6. [Ensure and Gather](book/06-ensure-and-gather.md)
7. [Verify the Invariant, Not the Roster](book/07-verify-the-invariant.md)
8. [Isolation for Parallel Writers](book/08-isolation-for-writers.md)
9. [Discovery Patterns: Loop-Until-Dry and Multi-Modal Sweep](book/09-discovery-patterns.md)
10. [Orchestration Shapes: Pipeline vs Barrier](book/10-orchestration-shapes.md)
11. [The Synthesis Step](book/11-the-synthesis-step.md)
12. [Prototype Before You Abstract](book/12-prototype-before-abstraction.md)
