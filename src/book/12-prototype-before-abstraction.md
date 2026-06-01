# 12. Prototype Before You Abstract

*Write the throwaway script first. The shape of the real feature lives inside it.*

Every orchestration system eventually needs a reconciliation verb — something that looks at the live state of a running agent team, compares it against the declared intent, and closes the gap. The temptation is to design this as a feature: sketch the API surface, debate the data model, open a ticket, wait for the right sprint. Resist it. Build a thin shell script over the primitives you already have. Run it against the real system. Watch it break.

The breaks are the design.

## Why a Script, Not a Design Doc

A design doc answers the question you know how to ask. A running script answers the questions the system actually has. These are not the same set.

When you write a reconciliation loop as prose — "fetch current state, diff against declared state, issue corrective commands" — the logic looks complete because the words are complete. The moment you script it, you discover that "fetch current state" means parsing output from three separate tools in three different formats, that the diff step silently swallows missing keys, and that "issue corrective commands" has a sequencing constraint nobody mentioned because everybody assumed somebody else had thought about it.

None of this appears in a design doc because design docs are written by humans who already understand the system. The script is written by a human who has to make the system do a specific thing by Tuesday, and that pressure surfaces every assumption.

## The Five-Step Order

There is a sequence that works. Do not reorder it.

**Declare the team.** Before any script runs, the team must exist as committed text: a charter file that names the roles, an engine map that assigns each role to a model tier, a live-state file that records what is actually running right now. Three files, all readable, none implicit. This is the ground truth the reconciliation loop will read from and write to.

**Script the reconciliation.** Write a shell script — fifty to two hundred lines, no more — that reads those three files, calls the orchestration tool's existing primitives (spawn, stop, status), and prints what it did. No new abstractions. No helper libraries. Every operation visible on one screen.

**Run it on the real thing.** Not a staging environment, not a sandbox with mocked tool calls. The actual machines, the actual agent runtime, the actual network. This is the step most teams skip. It is the only step that matters.

**Collect the failures.** Each failure is a free design review. A missing dependency, an ambiguous exit code, a race between spawn and status — these are the edge cases your abstraction will need to handle. You now have them enumerated, not theorized.

**Promote the proven logic.** Once the script runs cleanly against reality, lift its logic into the tool as a native verb. Same logic, fewer keystrokes, callable from anywhere. The abstraction is not a redesign — it is the script with the rough edges filed off and the hard-coded paths replaced by configuration.

## The Proof: A Missing Tool, Found in Five Seconds

During a session of building reconciliation logic for a five-agent deployment team, a script was written that read the live-state file, identified agents that had drifted from their declared configuration, and issued corrective spawn commands. The script was clean. The logic was correct. It had been reviewed by two people.

It was run on an actual host for the first time.

It failed immediately. A standard JSON parsing utility — present on every development machine, assumed to be universally available — was not installed on the production host. The script called it inline to extract a field from the live-state file. One missing package.

The fix took five seconds: swap the JSON call for a one-line shell alternative that extracts the same field using only POSIX tools. But the lesson was not about the fix. The lesson was about what would have happened if the reconciliation logic had been promoted to a native verb before that script ran.

The abstraction would have encoded the assumption. Every host in the fleet would have needed the utility installed. The dependency would have propagated silently into the tool's requirements, undocumented, until the next machine it ran on surfaced the same failure — but now buried inside a compiled binary instead of a readable script, and two abstraction layers away from the call site.

Because the script ran on the real thing, the assumption was visible. Because the assumption was visible, it was fixed. The native verb that was eventually promoted from that script has no external JSON dependency. It never did, because the prototype told the truth before the abstraction had a chance to lie.

## What the Prototype Teaches That Diagrams Cannot

A diagram of a reconciliation loop has no failure modes. A running script has exactly the failure modes the environment produces. These are the only failure modes that matter.

The prototype also teaches the latency profile. A reconciliation loop that looks instantaneous on a whiteboard may take twelve seconds when it has to poll five agents over a mesh VPN and wait for their status responses. You will not know this until you run it. Twelve seconds changes the design: you parallelize the polls, or you cache the last known state, or you accept the latency and document it. Any of these is a valid choice. None of them appear in the design until the prototype makes the time visible.

The prototype teaches the output format. When you watch the script print its actions to a terminal, you immediately know whether the output is useful. Too verbose, and operators ignore it. Too terse, and failures are invisible. You tune this in minutes with a script. You submit a ticket and wait three weeks to tune it in a shipped feature.

## When to Promote

The prototype is ready to promote when it has run successfully on real infrastructure more than once, when its failures are understood and handled, and when running it a second time produces no surprises. Not when the code is clean — it will never be clean. When the behavior is predictable.

Then lift the logic. Name the verb. Write the thin wrapper that calls the same sequence with the same guards and the same fallbacks, minus the scaffolding. The abstraction earns its existence by making a proven thing easier to call.

**The rule:** build the script before you build the feature, and run the script on the real system before you trust the script.