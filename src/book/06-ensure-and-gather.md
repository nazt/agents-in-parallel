# 6. Ensure and Gather

*Two idempotent verbs are all you need to keep a local agent team honest.*

---

The daily friction of running a local agent team is not configuration, not model selection, not prompt engineering. It is drift: the gap between the team you declared in a file and the processes that are actually running on your machine. Agents crash. You close a terminal. A relaunch lands in the wrong window. Within an hour of a fresh spawn, the declared team and the live team have diverged, and you are flying blind.

The fix is two orthogonal, idempotent verbs: **ensure** and **gather**.

---

## Ensure: Close the Lifecycle Gap

Ensure answers one question: for every member declared in the live-state file, is a live session running?

The algorithm is a three-way classify-and-act loop. For each declared member, probe the agent runtime for a session with that member's identifier. The result is one of three states:

- **Live** — session exists and is responsive. Skip. Do nothing.
- **Missing** — no session exists at all. Spawn a new isolated session with that member's config.
- **Dead** — a session record exists but the process has exited. Relaunch in place, reusing the existing session slot so the identifier stays stable.

That three-way branch is the entire ensure operation. Notice what it does not do: it never touches a live session. If you run ensure on a healthy, fully-staffed team, zero processes are disturbed. That is the idempotency guarantee. You can run ensure from a cron job, from a pre-task hook, from the top of every orchestration script — the cost on a healthy team is a few milliseconds of probe round-trips.

The missing-versus-dead distinction matters. Missing means spawn fresh: a new session, new environment, new working directory per the charter. Dead means the slot existed — the orchestration tool may still hold metadata, scroll buffer, window position — so you relaunch inside it rather than creating a duplicate. Skipping that distinction causes duplicate entries in the registry, which then require manual cleanup.

Spawning must be isolated. Each member gets its own session, not a pane in a shared window. The reason is robustness: a shared window means a single terminal multiplexer crash takes out multiple agents. Isolated sessions fail independently.

---

## Gather: Compose Without Restarting

Ensure gives you a correctly staffed team. Gather gives you a view of it.

The default state after ensure is a set of isolated sessions scattered across detached terminal multiplexer windows — invisible, uncoordinated to look at. Gather pulls them into your current window as a tiled layout of panes. Scatter is the inverse: it breaks the panes back out into isolated windows, returning the team to the detached state you started from.

The key mechanism: relocating a running pane is a **move**, not a restart. The process keeps executing through the relocation. A coding agent mid-way through a file edit does not notice that its pane was joined to a layout. This makes placement operations essentially free, and it makes gather re-runnable at any time — you can gather, inspect, scatter, and gather again without disrupting any agent's work.

Two failure modes bite you if you implement this naively.

**Address invalidation.** When a pane moves from its original window into your gather window, the old address — the window-and-pane index the orchestration tool used to target that member — is no longer valid. Gather must re-resolve the address of every relocated pane and update the registry entries accordingly. At the end of a gather operation, print the current send-command for each member so downstream scripts do not cache stale addresses. Scatter has the same obligation in reverse.

**Window name loss.** When you break a pane out of a window into a new standalone window, the terminal multiplexer assigns it a default name, usually the current process name or a generic placeholder. The original window name, which encodes the member identifier, is gone. The fix is to capture the window name before any break-out operation and re-apply it immediately after. Two lines of code, easy to miss, painful to debug — you end up with five windows all named "zsh" and no way to know which agent is which without reading process output.

---

## Composing the Verbs

Ensure and gather are orthogonal by design. Ensure is pure lifecycle; it knows nothing about pane layout. Gather is pure placement; it assumes all sessions already exist. That separation means each verb stays simple and each can be tested independently.

They compose cleanly: a flag on ensure — call it `--gather` — runs gather immediately after the lifecycle pass completes. This is the one-shot "make my team real and put it in front of me" command that you end up running at the start of every session.

A concrete example of this in practice: a five-agent health-probe team, each agent responsible for a different service cluster, runs autonomously during off-hours. In the morning, two agents have crashed on timeout errors, one is missing entirely because the session was never relaunched after a machine sleep, and two are live. Running `ensure --gather` spawns the missing agent, relaunches the two dead ones, leaves the two live agents untouched, then tiles all five into the current window. Total elapsed time: under four seconds. No agent that was mid-task was interrupted.

---

## Keeping the Verbs Honest

Both ensure and gather must read from the live-state file, not from memory or cached state. The live-state file is the declared reality; everything else is derived. If you start caching member lists in your orchestration script, you will eventually run ensure against a stale list and miss a member that was added by another session.

Ensure should log its actions — spawned, relaunched, skipped — at a single line per member. Silence on a live member, a line on any gap. That log is the audit trail that tells you, the next morning, what the team looked like when it reconvened.

Gather should be fast enough to run reflexively. If it takes more than a second on a team of ten, the address-resolution step is probably doing unnecessary round-trips. Probe once, batch the moves, re-resolve once.

---

**The rule:** Separate lifecycle from placement — ensure closes the gap between the file and the running process, gather composes the running processes into a view, and neither verb ever restarts what does not need restarting.