# 11. The Synthesis Step

*Fan-out is easy. Synthesis is where you earn the result.*

Spinning up N agents in parallel is a solved problem. Give each one a prompt, a tool set, and a scope; wait for the futures to resolve; collect the results. Any competent orchestration layer can do this. The part that fails — quietly, in ways that look like success — is what happens next.

Most teams treat synthesis as concatenation. They append the N responses into a single document, maybe sort by confidence score, hand it to the user. This is not synthesis. It is a pile wearing a trench coat.

## Contradiction Is a Signal, Not a Noise Floor

When two agents return different answers to the same question, the naive responses are:

- Average the results (nonsensical for non-numeric claims)
- Take the majority vote (correct on statistics, wrong on diagnosis)
- Pick the answer that matches your prior expectation (confirmation bias dressed as engineering)

All three discard the most valuable information the run produced: the fact that two agents diverged at all.

Consider a five-agent health probe. Four agents report a host reachable; one reports it down. Majority vote says the host is up. But the one dissenting agent used a fully-qualified domain name while the other four used a raw IP address. The contradiction is not noise — it is a directed pointer at a broken name-resolution layer. The synthesis step that votes it down as an outlier has just hidden an incident.

The right behavior: flag the divergence, inspect the mechanism each agent used, surface "these two results are not comparable because they exercised different paths." That is thinking. That is synthesis.

## The Compile Step Must Think

Think of synthesis as a compiler pass, not a text merge. A compiler doesn't concatenate object files; it resolves symbols, detects conflicts, and refuses to proceed if the graph is incoherent.

A synthesis agent's job is:

**Deduplication.** Multiple agents will surface the same finding through different framings. "The API returns 503 under load" and "the endpoint becomes unreachable at high concurrency" are the same claim. Merge them into one finding with both sources noted. Concatenating both creates false weight — the reader infers two independent observations when there is one.

**Conflict resolution with trace.** When two claims genuinely contradict each other, the synthesis step must not choose silently. It must record: what each agent found, what method it used, what source it read. Often the conflict resolves mechanically once the methods are compared (as in the DNS case above). When it doesn't resolve, the output should say so explicitly — "two agents reached opposite conclusions, here is why each is credible, here is what a tiebreaker run would need to examine."

**Hierarchy of evidence.** Not all agent results carry equal weight. An agent that read the primary source outranks an agent that inferred from secondary signals. The synthesis step should encode this: primary beats inferred, observed beats assumed, recent beats stale. A flat concatenation treats everything as equally authoritative.

**Single merged truth.** The final output should read as if one careful analyst did all the work. Not "Agent 3 says X, Agent 5 says Y." The attribution lives in footnotes or a provenance block; the main body presents the merged conclusion. If you cannot write a coherent merged conclusion, that is itself a finding: the run was insufficiently convergent and needs another round.

## The Completeness Critic

Even a well-executed synthesis can be complete within its own frame while missing the frame entirely.

Add one more agent after synthesis: a completeness critic. Its prompt is not "is this correct?" It asks:

- What modality was never exercised? (No agent tried the out-of-band path, the fallback endpoint, the read replica.)
- What claim was stated but never verified? (The synthesis says "this service is stateless" — did any agent actually check for session state?)
- What source was never read? (The charter file was summarized by an agent that didn't open it.)
- What adversarial case was never attempted? (Every probe was sent from the same network segment.)

The completeness critic does not produce a result. It produces a list of gaps. Each gap becomes a work item for the next fan-out round. The loop closes.

This is the step most systems omit, because it requires acknowledging that the run was incomplete — which feels like failure. It is the opposite. A run that knows its own boundaries is more valuable than a run that confidently covers three-quarters of the problem space while presenting itself as comprehensive.

## A Concrete Shape

A reliable synthesis pattern for a coding agent team looks like this:

1. **Fan-out**: each agent works a scoped problem and writes a structured result — findings, confidence, method, gaps it noticed.
2. **Collect**: wait for all results; record which agents timed out or errored (their absence is data).
3. **Synthesize**: one agent (or a deterministic function for simple cases) deduplicates, flags conflicts, traces each conflict to mechanism, produces a merged findings document with a provenance block.
4. **Critic pass**: one agent reads the merged document and the original task statement, then lists what is missing — no access to the original agent outputs, only to the synthesis and the task.
5. **Branch**: if the critic finds critical gaps, open a second fan-out targeting only those gaps; otherwise deliver.

Steps 3 and 4 are cheap — one or two model calls. The cost of skipping them is a subtly wrong answer that looks authoritative.

## Why This Is Hard to Build

The temptation is to compress synthesis into the prompt of the final summarizing call: "Here are N results, summarize them." This works until it doesn't — until a contradiction slips through as a consensus, until a gap in coverage is smoothed over by confident prose, until the reader acts on a finding that two agents would have resolved to the opposite conclusion if you had compared their methods.

Synthesis is a structured process, not a single LLM call. It has typed inputs (findings with method metadata), typed outputs (merged truth plus provenance), and a separate critic with a separate prompt that has no stake in validating the previous step's work.

Build it as a stage, not an afterthought.

---

**The rule:** treat every contradiction between agents as a pointer at a mechanism difference, force the synthesis step to think rather than concatenate, and always run a completeness critic whose only job is to find what the run didn't cover.