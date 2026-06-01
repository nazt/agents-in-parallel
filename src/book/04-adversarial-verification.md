# 4. Adversarial Verification

*Don't trust a finding because it is plausible; spend agents trying to kill it.*

---

Plausibility is not evidence. An agent that returns a confident diagnosis has done one thing: it has found an explanation consistent with the observable facts. That is not the same as finding the correct explanation. The gap between those two is where production incidents live.

Adversarial verification is the practice of allocating agents not to confirm a finding but to refute it. The structure is deliberate: the default verdict is "refuted," and the burden of proof falls on the original claim. If a majority of the verification agents cannot be convinced, the finding dies. This feels wasteful until you watch it catch the third error the authoring agents missed.

## Refute-by-Default Skeptics

The most common mistake in multi-agent verification is running N identical reviewers and averaging their confidence scores. You get correlated noise. If the first agent made a category error, the next four will make the same one because they are reading the same evidence with the same priors.

Invert the stance. Each skeptic's job is not to evaluate whether the claim is true; it is to find one specific reason the claim is false. Assign the verdict to "refuted" at initialization. Flip it to "confirmed" only if the skeptic exhausts its lens without finding a refutation. Aggregate across skeptics by majority: if more than half cannot refute it, the claim survives.

The consequence of this framing is that a single strong refutation from one narrow lens can kill a finding that nine other agents rated as probable. That asymmetry is intentional. Irreversible actions — publishing, deploying, sending — should require a high bar. Reversible actions can tolerate a lower one.

## Perspective-Diverse Verifiers

Diversity of stance must be structural, not requested. Asking agents to "consider multiple perspectives" in the system prompt does not produce diversity; it produces a single agent hedging. Diversity requires that each verifier receive a distinct failure-lens as its sole mandate.

Useful lenses for a software context:

- **Correctness**: Does the output match the specification under the inputs given?
- **Security**: Does this change introduce an injection surface, a privilege escalation path, or an information leak?
- **Reproducibility**: Can I follow these steps on a clean environment and arrive at the same result?
- **Regression**: Does this break anything that was previously true?
- **Boundary conditions**: What happens at the edges — empty input, maximum cardinality, clock rollover?

Five agents, five lenses. Each one is a specialist with a short brief and a binary output. The orchestrator collects verdicts and applies the majority rule. An agent that returns "cannot refute" after genuinely trying is contributing real information. An agent that returns "refuted" with a specific mechanism is contributing more.

## The Single-Mandate Review Gate

Before any irreversible action, insert a gate: one agent whose only job is to scan for a single, well-defined failure class. Not a general review. Not a quality pass. A narrow, specialized search.

The proof is concrete. A team of writing agents produced a document intended for public release. The authoring agents, the structural reviewers, and the style editors all passed it. Before publication, a single gate agent received one instruction: find any string in this document that looks like a private identifier — an internal username, a server hostname, an API key fragment, a path beginning with a company-internal prefix. It found three. The authoring agents had paraphrased around them, reducing their visibility, but had not eliminated them. The gate agent, running nothing but a pattern-and-heuristic scan with that single mandate, caught what the broader review missed.

The lesson is not that authoring agents are careless. The lesson is that agents optimizing for quality of prose are not simultaneously optimizing for absence of leaks. Those are different objective functions. Give each objective to a different agent.

## An Agent That Killed Its Own Hypothesis

The more instructive proof is when the verifier turns its skepticism on itself.

A five-agent health probe was investigating a silent failure in a distributed service mesh. One agent formed an early hypothesis: the failure was a name-resolution problem. The service could not find its upstream. The agent built a case — logs showed connection timeouts, which are consistent with DNS failures; the deployment had changed network topology recently; other failures in the fleet had been caused by resolution misconfiguration.

A well-incentivized agent treats its own hypothesis as a claim subject to the same adversarial treatment as any other. This one did. It ran resolution checks directly. Resolution succeeded. Every query returned the correct address with normal latency. The original hypothesis was not just unproven — it was actively falsified by the agent's own test.

Rather than preserving the explanation, the agent marked the hypothesis refuted and re-scoped. It looked for what else could produce connection timeouts when addressing was correct. It found a probe with no retry budget. The upstream was intermittently slow during cold starts; the probe hit the slow window, got a timeout, and reported failure. One-line fix.

If the agent had been built to defend its initial claim — if its reward signal came from having a hypothesis confirmed rather than from finding the true cause — it would have spent cycles explaining away the successful resolution test instead of discarding the hypothesis. The incentive structure determines whether an agent can discard its own work. Build it so that the honest answer, even when it is "I was wrong," is the rewarded answer.

## Building the Incentive Right

Adversarial verification is not a prompting trick. It is an architectural choice. You need agents whose output is a binary verdict with a mechanism, not a confidence score with hedges. You need a majority rule that gives a single strong refutation the power to kill a finding. You need gate agents so narrow in mandate that they cannot rationalize away the one thing they are looking for. And you need authoring agents that are not the same agents doing the verification.

The agent that writes the code should not be the agent that reviews it for security. The agent that formed the hypothesis should not be the one that tests it — or if it must be, the test must be designed to falsify, not to confirm.

**The rule:** Default the verdict to refuted, diversify the lenses, and build every agent so that "I was wrong" is as valid an output as "I was right."