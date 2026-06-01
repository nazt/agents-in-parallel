# 8. Isolation for Parallel Writers

*Two agents, one directory, zero conflict markers — and half your work gone.*

Parallel agents that only read can share whatever they like. Point three search agents at the same repository, the same log directory, the same knowledge base — they will never interfere. Read operations commute. Write operations do not.

The moment two agents write to the same working directory on the same branch, you have a clobber trap. Not a merge conflict. Not a clear error. A clobber trap is silent: the second writer's flush overwrites the first writer's, the file system records one version, and nobody complains. The only evidence is the missing work.

## The Mechanism

A version-control working directory is a mutable snapshot. It holds one checked-out state at a time. When an agent edits a file, it modifies that snapshot in place. When a second agent edits the same file in the same directory, it modifies the same snapshot. Whichever process writes last wins. There are no conflict markers because there was no merge — just two sequential writes to the same inode.

The fix is structural, not procedural. You cannot solve this with careful scheduling or polite conventions. You solve it by giving each writing agent its own isolated working copy, on its own branch. Git worktrees are the clean primitive for this: a single repository can have multiple checked-out working trees simultaneously, each on a distinct branch, each in a distinct directory on disk. Writers cannot collide across worktree boundaries. The isolation is enforced by the file system, not by agent discipline.

## Proof: The Shared-Directory Trap

A convenience spawn brought up a coordination agent plus two coding agents. The spawn was fast — all three appeared in a single command, roster printed cleanly, team announced ready. Coordination in one pane, Coder A in another, Coder B in a third.

All three landed in the same working directory, on the same branch.

The coordination agent was fine. It read tickets, wrote summaries to a scratch buffer, never touched source files. Read-only agents can share anything. The problem was Coder A and Coder B. Each had been handed a task that required editing source files. The moment both agents started writing — not if, when — one would overwrite the other. The file system would pick a winner based on flush timing, and the loser's work would disappear without a trace.

The team looked correct from the outside. Three members, three roles, a printed roster saying each member was ready and assigned. The declaration of readiness was accurate for the coordination agent. It was a fiction for the coders.

## Why You Cannot Patch a Live Process's Birthplace

The obvious repair is to relocate the offending agents into their own worktrees. The problem: you cannot move a running process's working directory after the fact. A shell process inherits its working directory at spawn time. Its current directory is a kernel attribute of that process, and you cannot retroactively change it from outside the process — not cleanly, not portably, not safely for a live agent that may be mid-operation.

The repair for the shared-directory trap is therefore destructive: you stop the incorrectly spawned agents, provision separate worktrees, and respawn each writer into its own isolated ground. There is no patch-in-place option. This is not a flaw in the tooling; it is a consequence of how operating systems model process working directories.

This means isolation must be declared before spawn, not fixed after. The worktree must exist, the branch must be created, and the spawn command must target the correct directory — in that order, before the agent process starts.

## The Proof Requires Observation, Not Trust

Declaring isolation in a config file or a spawn script is a request. It is not a guarantee. The only guarantee comes from observing the running system.

After a team of writers comes up, verify two properties before declaring the team ready:

**One: distinct working directories.** Enumerate each agent's working directory — query the process, the orchestration tool's agent registry, or whatever source of truth your runtime exposes. Assert that no two writing agents share a path. If two paths are identical, the team is not safe, regardless of what the config says.

**Two: distinct branches.** List the branch each worktree is checked out on. Assert uniqueness among writers. Two agents in separate directories but on the same branch is still dangerous if you later run a merge step that assumes branch-per-agent isolation.

A three-line verification that checks these two properties is worth more than any amount of spawn-time convention. Run it after every team bring-up. If the assertion fails, stop the team before any writes happen. Clobber damage is hard to detect and harder to recover; preventing it costs almost nothing.

## Shared Directories Are Fine — for Readers

This is not a rule that all agents must be isolated from each other. Read-only agents — reviewers, searchers, report generators, anything that never modifies a file — can share a working directory freely. There is no cost to giving them a common checkout. The isolation requirement is scoped precisely: **one worktree per writing agent, no exceptions**.

A team of five where two write and three read needs two isolated worktrees (one per writer) and can share or not share for the readers. The constraint is proportional to the hazard.

## Structure of a Safe Writer Spawn

The sequence that avoids the trap:

1. Create the branch for this agent's work.
2. Add a worktree for that branch in a dedicated directory.
3. Spawn the agent with its working directory set to that worktree path.
4. After spawn, verify: working directory matches the expected path, branch matches the expected name.
5. Record both in the team's live-state file so later stages can locate each agent's output.

Steps 1–3 are the setup. Steps 4–5 are the proof. Skip the proof and you are trusting declarations, which is how the shared-directory trap happens in the first place.

## Summary

Shared state is safe for readers and fatal for writers. The structural fix is git worktrees — one per writing agent, provisioned before spawn. The process-working-directory constraint means you cannot repair this after the fact; you destroy and recreate. And a declaration of isolation in config is only a request: close the loop by observing the actual running directories and asserting uniqueness.

**The rule:** every agent that writes gets its own worktree and its own branch, provisioned before spawn and verified after — declaration is not proof, observation is.