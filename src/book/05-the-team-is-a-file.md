# 5. The Team Is a File

*A running team of agents looks like a process; treat it like a document and you get reproducibility for free.*

---

Every team of parallel agents encodes three categories of fact. The mistake most builders make is treating all three facts as one thing — dumping them into a single config blob and calling it done. The categories change at different speeds, are owned by different parties, and have completely different durability requirements. Conflating them is why teams are hard to reproduce and easy to lose.

Name the three categories and the architecture becomes obvious.

---

## WHO and WHAT: The Charter

The charter is a human-authored recipe. It names the roles on the team, describes the intent of each role, and sets the boundaries of what the team is for. A coding-review team might have a proposer, a critic, and a summariser. A five-agent health probe might have one scout per service endpoint and an aggregator that waits for quorum. The roles are stable. They change when the human decides the team's job has changed — not when a process crashes, not when you switch machines, not when you upgrade a runtime.

Because the charter encodes *intent*, it is irreplaceable. You cannot reconstruct it from a snapshot of running processes. If you lose the charter, you lose the knowledge of why the team exists and what invariants it is supposed to uphold. This is the fact that must live in version control.

The charter is also the only file a human should need to read to understand what a team does. Keep it short. One role per stanza, one line of intent per role, explicit preconditions if any. It is a recipe, not a manual.

---

## HOW: The Engine Map

The engine map is a dictionary. Each entry maps a friendly runtime name — a short token you use throughout the charter and the orchestration tool — to a literal launch command: the shell invocation, the flags, the environment variables that boot exactly one agent in exactly one runtime.

The engine map is the only cross-runtime file in the system. One entry might boot a frontier model from vendor A; the next might boot a different vendor's runtime entirely. The map is what lets you write a charter that says `role: critic, engine: fast-reviewer` without encoding a vendor decision into the charter itself. You swap the engine map entry and the charter stays clean.

Like the charter, the engine map belongs in version control. It changes when you decide to switch runtimes or tune launch parameters — a deliberate human decision, not a runtime event.

---

## NOW: The Live State

The live state is written by the system, not by a human. When the orchestration tool boots the team, it records the pane identifiers assigned by the terminal multiplexer, the process IDs, which members are currently alive, their working directories, any inter-agent message-bus addresses. This is a snapshot of *this instantiation* of the team on *this machine at this moment*.

The live state is meaningless anywhere else. The pane identifiers are local to one multiplexer session. The process IDs are local to one OS. The working directories are absolute paths on one filesystem. Committing the live state to version control is noise at best and misleading at worst — a teammate on a different machine would get a file that looks authoritative but describes a world that no longer exists.

The live state is also fully derivable. Given a charter and an engine map, the orchestration tool can re-boot the team from scratch and write a fresh live state. The derivability is the entire point: it means you can treat the live state as a disposable cache, not a source of truth.

---

## The Proof: A Dead Snapshot

A team of five coding agents was running a long refactor across parallel worktrees. The terminal multiplexer session died — power loss, not a graceful shutdown. The live-state file survived on disk, listing pane IDs, process IDs, agent statuses.

Re-attaching to those pane IDs failed silently: the multiplexer session was gone, so the IDs referred to nothing. Attempting to send a message to the recorded process IDs produced no output and no error. The live-state file looked like a map, but every address on it was a dead letter.

Recovery took thirty seconds. The orchestration tool read the charter and the engine map — both in version control, untouched — and re-booted the team into a fresh multiplexer session. The new live-state file was written, the agents picked up their worktrees, and the refactor resumed. Nothing was lost except the in-flight work that had not been committed, which is a git problem with a git solution.

If the charter had not been in version control — if it had been stored only in the live-state file, or in the session state of the multiplexer — recovery would have required reconstructing intent from running processes that no longer ran. That is not a recoverable situation.

---

## The Asymmetry Is the Architecture

The three files have different volatility signatures. Charter: low volatility, human-authored, git-committed. Engine map: medium volatility, human-decided, git-committed. Live state: high volatility, machine-written, git-ignored.

The asymmetry is not an accident of implementation. It is the load-bearing property of the design. You can always regrow the live state from charter plus engine map. You can never reconstruct the charter from a dead snapshot. That single directional dependency — live state derives from the other two, not the other way around — determines what to commit and what to discard.

When you add a new role to the team, edit the charter. When you swap runtimes, edit the engine map. When the team crashes and you need to restart, delete the live-state file and let the orchestration tool regenerate it. Three files, three ownership models, three commit policies.

---

**The rule:** commit what humans author and machines cannot reconstruct; discard what machines write and can always regenerate.