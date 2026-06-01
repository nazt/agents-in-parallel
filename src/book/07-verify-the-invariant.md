# 7. Verify the Invariant, Not the Roster

*The declaration tells you what you intended; the running process tells you what exists.*

You've assembled the team. The registry shows five agents, each with a name, a role, and a status light that reads "active." The charter file lists their responsibilities. The engine map says which runtime each one runs. Everything looks correct. You are about to begin distributing work.

Stop. You have verified the roster. You have not verified the invariant.

The distinction matters more than it first appears, and the cost of ignoring it compounds with every agent you add.

## Declarations Are Not Evidence

A roster is a record of intent. It describes what should exist: which agents were spawned, which directories they were assigned, which runtimes they were handed. A declaration can be perfectly accurate at the moment of creation and silently wrong ten seconds later. The agent crashes. The pane survives. The status field doesn't know.

The invariant is the property you actually need: each agent is running its designated runtime, in its designated directory, processing work. That property must be read from the running world, not inferred from the config that described the world before it started.

This isn't a paranoid edge case. It's the normal failure mode of any system that hands you a view derived from events rather than from continuous inspection.

## The Liveness Trap

Every terminal multiplexer ships an activity field. It is typically computed from recency: if the pane produced output within some window, it's "active"; beyond the threshold, it degrades to "idle" or "stale." This field is the obvious proxy for liveness and it is wrong.

Consider a coding agent deep in a long compilation or waiting on a slow API call. No output. Six minutes pass. The multiplexer marks the pane stale. That agent is alive, working, blocked on I/O — and indistinguishable, by activity timestamp alone, from a pane where the runtime exited and dropped to a bare shell. The activity field conflates "not printing" with "not running." Those are different facts.

The reliable signal is the pane's current foreground process. Query it directly. If the foreground process is the agent runtime, the agent is alive. If the foreground process is a bare shell, the runtime exited. That distinction does not fade with time. It does not lie about agents that are thinking quietly. It is a present-tense fact about the operating system's process table, not a timestamp derived from past behavior.

The implementation is straightforward: for each pane, resolve its foreground process group, walk the process tree, and check the executable name against the expected runtime. Five lines of shell. If it matches, the agent is live. If it doesn't, the agent is gone regardless of what the status field says.

## The Silent Clobber

Activity status is the obvious failure. Working directory is the subtle one, and it can destroy correctness while the roster looks perfectly healthy.

Here is the proof as it happened. A team of three coding agents was assigned to work in parallel, each in a separate git worktree. The worktree pattern is the correct approach: isolated checkouts, no shared index, no accidental cross-contamination. The orchestration tool reported all three agents active. The layout looked tidy. Status: healthy.

A five-point health probe queried each agent's actual working directory by reading the process's current directory from the operating system — not from the config, not from the spawn command, from the live process. All three agents reported the same path. They were all working in the same worktree.

The spawn logic had a bug: the directory argument was being constructed before the worktrees were fully initialized, and all three resolved to the same default. The orchestration tool didn't notice because it never checked. The roster showed three distinct entries. The world contained three agents clobbering each other's work in one directory, racing on every write.

The roster lied. The working directory couldn't.

## What to Inspect

Given that declarations are not evidence, what constitutes a real health check before you begin distributing work?

**Process identity.** Is the expected runtime the foreground process? Not "was it launched," not "has it printed recently" — is it running now?

**Working directory.** Is each agent in the directory it was assigned? This check takes one syscall per process. It catches the silent clobber. It costs nothing relative to the work you're about to hand out.

**Uniqueness.** If each agent is supposed to be isolated — in its own worktree, its own port, its own namespace — verify that the values are distinct across the team. A set of five working directories that collapses to three unique paths means two pairs of agents are sharing space they shouldn't.

**Reachability.** If agents expose an interface — a local HTTP endpoint, a socket, a named pipe — verify you can complete a round trip. A process can exist and be unreachable. A reachable endpoint proves more than a running process.

These are invariants. They describe properties the system must have for the work to be correct. Verifying them is not overhead; it is the proof that the system is in the state the roster claims it's in.

## Inline, Not Postmortem

The natural instinct is to run these checks after something goes wrong. That's postmortem debugging, and by then you've distributed work into a broken environment, paid the cost of the failure, and spent time diagnosing something that was verifiable before you started.

Structural verification belongs in the startup sequence, before the first task is dispatched. It takes seconds. It converts a class of silent failures — wrong directory, dead runtime, duplicate assignment — into loud, early, actionable errors. The orchestration loop doesn't need to be clever. It needs to ask the operating system what's actually true, compare that to what should be true, and refuse to proceed if they don't match.

Announcements of readiness are cheap. Proof of readiness is a process query.

**The rule:** before distributing work to any agent team, verify each invariant — live process, correct directory, unique assignment, reachable interface — by inspecting the running system directly; a clean roster is necessary but not sufficient, and the gap between them is where silent failures live.