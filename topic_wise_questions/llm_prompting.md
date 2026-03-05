# Interview Questions: AI for Coding / Programming / Development

---

## Section 1: Fundamentals & Mental Models

---

**Q1. How do you think about AI coding tools — are they autocomplete on steroids, or something fundamentally different? Why does the distinction matter?**

**Sample Answer:**

They're fundamentally different, and conflating them with autocomplete is one of the most dangerous mental mistakes a developer can make. Autocomplete operates on syntax and local token prediction. It knows what *word* comes next. AI coding tools — when used well — operate on *intent*. They can understand that you're building a rate limiter, infer that you probably need Redis, recognize the pattern you're following from three files away, and generate code that fits your architecture.

The distinction matters because it changes *how* you interact with them. If you think it's autocomplete, you accept whatever it generates and move on. If you understand it as an intent-translation layer, you treat it as a collaborator — you give it richer context, you validate its reasoning, and you use it to *amplify* your architectural thinking rather than substitute for it. Engineers who treat it as glorified autocomplete get 10% productivity gains. Engineers who deeply understand what it is and isn't get 10x leverage on certain classes of problems.

The failure mode of misunderstanding this is subtle: your code starts looking locally correct but globally incoherent, because you've been accepting token-level suggestions without asserting system-level constraints.

---

**Q2. What are the categories of programming tasks where AI assistance provides the highest ROI, and where does it actively hurt you?**

**Sample Answer:**

Highest ROI categories:

- **Boilerplate and scaffolding**: CRUD APIs, CLI argument parsing, test setup, config files. Things that are well-specified, pattern-heavy, and low-stakes to get wrong.
- **Cross-language translation**: Porting a Python function to Go or Rust. The AI understands both semantic intent and idiom differences.
- **Test generation**: Given a function signature and implementation, generating edge case tests is something AI does remarkably well — often better than engineers, because it's unconstrained by familiarity bias.
- **Regex and format parsing**: One of the clearest wins. Describing what you want in English and getting a correct regex is faster and often more accurate than writing it manually.
- **Documentation**: Generating docstrings, README sections, API docs from code is extremely high ROI.
- **Unfamiliar library exploration**: "Show me how to use X library to accomplish Y" is a great use case, dramatically faster than reading docs cold.

Where it hurts you:

- **Novel architecture decisions**: AI will confidently produce architecturally plausible but deeply wrong system designs when the constraints are complex and non-obvious. It'll give you a microservices diagram when your team of three should be building a monolith.
- **Security-sensitive code**: It generates code that *looks* secure but often misses subtle vulnerabilities — off-by-one in buffer handling, JWT validation edge cases, SQL injection in dynamic query builders.
- **Performance-critical paths**: It optimizes for readability and correctness, not cache behavior, branch prediction, or memory layout. It'll give you a beautiful O(n log n) sort when you needed a radix sort for your specific data distribution.
- **Highly coupled legacy systems**: Without understanding the hidden contracts between components, its refactoring suggestions will break invariants silently.
- **Learning**: If you're a junior engineer trying to understand something, using AI to get the answer immediately is actively harmful to skill development. You're renting understanding instead of building it.

---

**Q3. How do you evaluate whether an AI-generated code suggestion is trustworthy without running it?**

**Sample Answer:**

I have a mental checklist that runs roughly in this order:

**1. Plausibility scan — does it match the shape of what I know to be correct?** Before reading it carefully, I look at structure. Does it handle the happy path, the error path, and edge cases? Is the error handling idiomatic for this language? Suspiciously clean code with no error handling is a red flag.

**2. Seam identification — where are the boundaries where things could go wrong?** I look for I/O boundaries (file reads, network calls, DB queries), type coercions, null/nil handling, and concurrency primitives. These are where AI tends to be confidently wrong.

**3. Invariant checking — does this code maintain the invariants I know this system requires?** If I know this function must always return a non-empty list, or that a database transaction must be rolled back on any exception, I verify those explicitly.

**4. Library API verification** — AI confidently uses APIs that don't exist or have changed. I cross-reference any non-trivial library call against documentation. This is one of the most common sources of subtle bugs.

**5. The "too clean" heuristic** — if the code handles exactly the case I described and nothing else, and it looks elegant, I get suspicious. Real robust code is usually a little ugly because it handles the cases you *didn't* ask about.

**6. Counterfactual testing** — I ask myself: "What input would break this?" If I can immediately think of one, that's a signal to harden the code before moving on.

The honest answer is that for anything non-trivial, I run it. But the above heuristics help me decide how much testing harness to build before I trust it.

---

## Section 2: Prompt Engineering for Code

---

**Q4. Walk me through how you'd prompt an AI to implement a non-trivial feature — say, a distributed rate limiter with Redis. What do you include and why?**

**Sample Answer:**

Bad prompt: *"Write a distributed rate limiter using Redis."*

That produces a textbook sliding window implementation that ignores my actual constraints.

Here's how I'd actually do it:

**Step 1 — Establish context.** I paste in the relevant parts of my existing codebase: the HTTP middleware interface, the Redis client wrapper I'm using, the config struct pattern we follow, and a representative existing middleware as a style example. This constrains the output to be *idiomatic for my codebase*, not generic.

**Step 2 — Specify the algorithm explicitly.** "I want a token bucket algorithm, not a sliding window." If I have a preference for why (predictable burst handling), I state it. AI defaults to sliding window because it's the most common in training data.

**Step 3 — State the failure mode contract.** "When Redis is unavailable, fail open and log the error. Do not fail requests." Or closed, depending on requirements. AI will pick one arbitrarily if you don't specify.

**Step 4 — State non-obvious constraints.** "Keys must expire. The Lua script must be atomic. The implementation must handle Redis Cluster keyslot requirements, so all keys for a given request must hash to the same slot."

**Step 5 — Specify the interface first.** I ask it to write the function signature and the Redis key schema first, and I review and correct those before it generates the implementation. This is the technique that separates good prompt engineers from bad ones — you checkpoint at the *contract* level before you get an implementation you have to throw away.

**Step 6 — Ask for tests separately.** After I've reviewed the implementation, I ask for tests in a fresh context window, giving it the implementation. Asking for implementation and tests together usually produces tests that just call the happy path.

The meta-principle is: **the specificity of your prompt determines the quality of the output, and the quality of the output asymptotes to the quality of your mental model of the problem.** AI can't fill in constraint gaps you don't know you have.

---

**Q5. What's the difference between a good prompt and a great prompt when asking AI to debug code?**

**Sample Answer:**

A good prompt: *"Here's my code, it's throwing a NullPointerException, fix it."*

A great prompt includes five things that a good prompt doesn't:

**1. The exact error output, not a paraphrase.** Stack traces are precise. "It's throwing a NPE" tells the AI very little. The full stack trace tells it exactly which line, what call chain, and often what value was null.

**2. What you've already tried.** This is the most underused technique. "I already added a null check on line 47 and the error moved to line 52" dramatically narrows the search space and prevents the AI from giving you advice you've already exhausted.

**3. The invariant you believe is being violated.** "This method should never receive a null `user` because the middleware guarantees authentication before this handler runs." This tells the AI where to look for the broken contract upstream.

**4. The minimal reproduction.** Pasting 500 lines and asking "what's wrong" forces the AI to do triage, and it often focuses on the wrong thing. Pasting 20 lines that isolate the problem is dramatically more effective.

**5. What the correct behavior *should be*.** "When the user has no profile photo, this should return the default avatar URL, not crash." This tells the AI what a correct fix looks like, which prevents it from fixing the crash by swallowing the exception.

The underlying principle is: **debugging is hypothesis narrowing, and your prompt should encode every hypothesis you've already falsified.** AI is a reasoning engine, not a magic oracle — if you give it more signal, it generates better hypotheses.

---

**Q6. How do you use AI for code review, and what are the risks of relying on it for that purpose?**

**Sample Answer:**

I use AI for code review in layers:

**First pass — automated**: I pipe diffs through an AI with a structured review prompt that asks it to look for: security issues, missing error handling, performance antipatterns, broken invariants stated in comments or nearby code, and missing test coverage for edge cases. This catches a class of issues that humans often miss because we pattern-match too quickly on code that *looks* familiar.

**Second pass — targeted**: For specific code I'm uncertain about — a new concurrency primitive, a cryptographic implementation, a performance-critical loop — I paste it with a focused question: "Is there a race condition in this code if `updateCounter` and `readCounter` are called from separate goroutines?"

Where it fails and where the risks are:

**Risk 1 — False confidence.** AI review is comprehensive-looking. It produces a bulleted list of concerns that feels like a thorough review. This creates a psychological tendency to then *under-review* the code yourself. The danger is that AI misses systemic issues — it reviews the code you gave it, not the system it lives in.

**Risk 2 — It can't see what's missing.** AI is good at finding problems in code that exists. It's bad at noticing that you forgot to invalidate the cache, or that this function needs to emit an audit log, or that the new endpoint is missing from the API spec. It can only review what you gave it.

**Risk 3 — It normalizes bad patterns.** If your codebase has a consistent antipattern — say, you're using exceptions for flow control — AI trained on your code will not flag that pattern, because it looks internally consistent.

**Risk 4 — Security review hallucinations.** AI will confidently tell you code is secure when it isn't, especially for subtle issues like timing attacks, TOCTOU vulnerabilities, or cryptographic misuse. I never use AI as a substitute for a human security review on sensitive code.

My policy: AI review is a first-pass filter, not a gate. It reduces the surface area for human reviewers, it doesn't replace them.

---

## Section 3: AI-Assisted Architecture & System Design

---

**Q7. How would you use AI to help you design a system you've never built before? And what guardrails do you put on that process?**

**Sample Answer:**

My approach is to use AI as a *thinking partner*, not an answer machine, and the distinction is critical.

**Phase 1 — Problem decomposition.** I describe the system requirements and ask the AI to identify the key decisions I need to make and the tradeoffs inherent in each. I'm not asking it *what to do*, I'm asking it to *enumerate the decision space*. This is where AI is excellent — it has seen many systems and can surface tradeoffs I haven't considered.

**Phase 2 — Devil's advocate.** For any approach I'm considering, I ask the AI to argue against it. "I'm planning to use a fan-out pattern for this notification system. What are the failure modes of this approach at 10 million users?" This stress-tests my thinking.

**Phase 3 — Prior art search.** "What are the known architectural approaches to this class of problem? What do companies at scale do?" This gives me a map of the solution space, which I then research independently.

**Guardrails:**

- I **never** let AI make the final call on any architectural decision. It doesn't know my team's operational capacity, my organization's risk tolerance, or my existing infrastructure constraints.
- I **always** verify any specific claim about performance characteristics, consistency guarantees, or operational behavior. AI confidently states wrong things about CAP theorem tradeoffs, Kafka retention semantics, and database locking behavior.
- I **discard** AI suggestions that don't come with a clear reasoning chain. "Use Kafka for this" with no explanation gets ignored. "Use Kafka because you need durable message replay across multiple consumer groups at different offsets" is a claim I can evaluate.
- I treat AI architectural suggestions the way I treat a smart colleague who's been out of industry for a year — smart, useful, but needs verification on specifics.

---

**Q8. Can AI help with technical debt reduction? How would you structure that workflow?**

**Sample Answer:**

Yes, but it requires a very specific workflow or you end up generating new technical debt faster than you're eliminating it.

**Identification phase:**

I feed the AI the codebase (or relevant modules) with a prompt focused on *categorizing* debt, not fixing it. I ask it to identify: dead code, inconsistent patterns for the same operation, abstraction inversions (where high-level modules depend on low-level details), functions that do more than one thing, and areas where error handling is inconsistent.

The output is a prioritized list. I don't act on it yet.

**Triage phase:**

I review that list with my team. AI often surfaces real issues but can't rank them by business risk or migration cost. A human has to own that prioritization. We apply the rule: *debt that's in frequently-changed code is worth paying immediately; debt in stable, rarely-touched code may not be worth touching.*

**Execution phase — small, isolated refactors:**

For each debt item, I give AI the specific module, the problem statement, and the *contract that must be preserved* — "This function must still return the same type and must not change its public API." I never ask AI to "refactor the whole service." I ask it to "extract this business logic from this HTTP handler into a separate struct."

**Verification phase:**

Every AI-assisted refactor must pass the existing test suite without modification. If tests need to change, that's a signal that either the AI changed behavior it shouldn't have, or the tests were testing implementation rather than behavior — which is itself a form of technical debt worth addressing explicitly.

The meta-risk: AI-assisted debt reduction without strong tests is actively dangerous. You're moving fast through code you don't fully understand. If you don't have tests, write them before you refactor. No exceptions.

---

## Section 4: AI Tooling & Ecosystem

---

**Q9. Compare the different paradigms of AI coding tools: inline completion (Copilot-style), chat-based (Claude/ChatGPT), and agentic (Devin/Claude Code style). When do you reach for each?**

**Sample Answer:**

These are genuinely different tools for different cognitive modes, and conflating them is a productivity mistake.

**Inline completion (Copilot-style):**
Best for *flow state* coding. When I know what I'm building and I'm just executing, inline completion reduces mechanical friction. It's highest value on: boilerplate, repetitive structural patterns, test assertions after I've written the first one, and filling in function bodies once I've written a signature that encodes my intent.

The risk is that it interrupts your thinking if you're not already in execution mode. If you're still *designing*, accepting inline completions anchors you to the first plausible implementation rather than the best one.

**Chat-based (Claude/ChatGPT style):**
Best for *deliberate problem solving*. When I'm stuck, exploring a new API, trying to understand a system, debugging a hard problem, or reviewing code I'm uncertain about. Chat allows me to have a conversation — to push back, ask follow-ups, and iterate on understanding.

I also use chat for anything where I need to provide substantial context — pasting multiple files, explaining a complex system constraint, or asking it to synthesize across several concerns at once.

**Agentic (Claude Code, Devin style):**
Best for *well-specified multi-step tasks* with clear success criteria. "Add pagination to all API endpoints that return lists. Use cursor-based pagination. Add tests. Update the OpenAPI spec." — this is a task that spans files, requires coherent execution of multiple steps, and can be verified by running tests and checking the spec.

The failure mode of agentic tools is using them for *poorly specified* tasks. "Improve the codebase" given to an agent produces noise. "Add input validation to all user-facing endpoints based on the existing validation patterns in `src/validators/`" produces value.

My mental model: inline completion is for execution, chat is for thinking, agents are for delegation. You need to be in the right cognitive mode to get value from each.

---

**Q10. How do you maintain code ownership and deep understanding of your codebase when AI is writing significant portions of it?**

**Sample Answer:**

This is one of the most important and underappreciated challenges in AI-assisted development, and most teams are failing at it silently.

**Technique 1 — The "explain it back" discipline.** Before accepting any AI-generated code that I didn't fully understand, I ask myself: "Can I explain to a colleague exactly why every line of this is the way it is?" If the answer is no, I either ask the AI to explain it to me, or I rewrite it in a way I do understand. The goal is: no mystery code in my codebase.

**Technique 2 — Mandatory code review for AI-generated code.** AI-generated code should never bypass your review process. At some teams there's an implicit "it came from AI, it's probably fine" culture that's very dangerous. AI code should be reviewed *more* skeptically than human-written code, not less.

**Technique 3 — Domain modeling by hand.** I've adopted a rule: data models, API contracts, and system interfaces must be designed by humans, not AI. These are the load-bearing walls of your software. AI can implement against them, but it can't design them well because it doesn't understand your domain.

**Technique 4 — Knowledge documentation as a first-class artifact.** When AI helps me understand or implement something non-trivial, I write an Architecture Decision Record or a comment that explains *why* the code is structured the way it is. This decouples team knowledge from "whoever interacted with the AI that day."

**Technique 5 — The debugging test.** Periodically, I will intentionally *not* use AI to debug a problem in AI-generated code. This forces me to actually understand what was written. If I can't debug it without AI assistance, I've lost ownership, and I pay the tax of understanding it properly.

The honest meta-point: if your team is generating code faster than it can understand, you're accumulating understanding debt. At some point, that debt comes due — usually during an incident at 2am.

---

## Section 5: AI, Testing & Quality

---

**Q11. How do you use AI to improve test coverage without just generating tests that test nothing?**

**Sample Answer:**

The core failure mode of naive AI test generation is: you ask for tests, AI writes tests that call the happy path and assert the obvious, your coverage metric goes to 80%, and you feel safe. You're not safer — you just have more code to maintain.

Here's how to get tests that actually catch bugs:

**Strategy 1 — Give it the implementation AND an adversarial role.** "Here is this function. Your job is not to write tests that pass. Your job is to write tests that *would catch a buggy implementation*. For each test, explain what bug it would catch if the implementation were wrong."

This reframes the task. Instead of generating confirming tests, it generates *falsifying* tests.

**Strategy 2 — Property-based test generation.** Ask AI to generate property-based tests rather than example-based tests. "What are the invariants that should hold for any valid input to this function?" Then implement those as property tests using QuickCheck, Hypothesis, or whatever your language supports. This generates much higher signal.

**Strategy 3 — Boundary condition enumeration.** Explicitly ask: "What are all the boundary conditions, edge cases, and error conditions for this function?" Get that list first. Then ask for tests that cover each item on the list. Don't let AI decide what to test — let it enumerate the cases, then you curate them, then it writes the tests.

**Strategy 4 — Mutation testing verification.** After AI generates tests, run a mutation testing tool. If mutants survive, you have tests that aren't actually catching anything. Feed the surviving mutants back to AI: "These mutations weren't caught by the test suite. Write tests that would catch each one."

**Strategy 5 — Chaos and adversarial input.** Ask AI to generate fuzz corpus entries or adversarial inputs specifically for parser code, serialization code, or any code that handles external data. It's excellent at this because it understands common attack vectors and malformed input patterns.

---

**Q12. How do you think about the role of AI in a CI/CD pipeline?**

**Sample Answer:**

AI in CI/CD is still early, but I see three meaningful integration points today and one aspirational one:

**Current: AI-powered code review as a PR gate.** Tools that run an AI review on every PR diff and require addressing critical findings before merge. The key design decision is: make it a *required check* for certain categories (security, obvious bugs) and an *advisory check* for style/quality issues. Teams that make all AI feedback blocking will kill developer velocity; teams that make none of it blocking get no value.

**Current: AI-assisted flaky test detection.** CI generates a lot of signal that humans don't analyze well — flaky test patterns, test execution time trends, correlation between code changes and test failures. AI can surface these patterns ("this test has a 15% failure rate and always fails when this module is modified"), which is genuinely hard to do with traditional tooling.

**Current: AI log analysis on failure.** When a build or test fails, AI can analyze the failure logs, correlate with recent code changes, and produce a root cause hypothesis. This cuts mean time to understanding dramatically. I've seen this reduce "why did CI fail?" investigation from 20 minutes to 2 minutes.

**Aspirational: AI-generated regression tests on merge.** The idea that AI could analyze a code diff and automatically generate regression tests that cover the changed behavior before merge. This exists in early forms but isn't reliable enough yet for most teams. When it works well, it closes the loop where developers add features without tests.

**The risk I'd flag**: AI in CI creates a pressure to optimize for what AI measures. Teams start writing code that passes AI review rather than code that's actually correct. You need to stay vigilant that you're using AI as a tool for quality, not gamifying its metrics.

---

## Section 6: Security, Ethics, and Responsible Use

---

**Q13. What are the security risks specific to AI-assisted development, and how do you mitigate them?**

**Sample Answer:**

This is a category of risk the industry is significantly underestimating right now.

**Risk 1 — Training data poisoning / supply chain attacks.** AI models are trained on public code, including malicious code specifically crafted to appear in training data and influence model outputs. There are documented cases of "indirect prompt injection" attacks where models trained on poisoned repositories produce subtly backdoored code. Mitigation: treat AI-generated security-sensitive code with the same scrutiny you'd apply to code from an unknown third party.

**Risk 2 — Confidently wrong cryptographic implementations.** AI generates cryptographic code that looks correct but uses deprecated algorithms, incorrect IV handling, insecure random number generation, or misunderstood padding schemes. Mitigation: use vetted cryptographic libraries and never ask AI to implement cryptographic primitives. Ask it only to *use* well-established libraries correctly.

**Risk 3 — Secret leakage through context.** Engineers pasting code into AI chat interfaces may inadvertently include API keys, database credentials, or internal system architecture. Even with privacy controls, the risk is real. Mitigation: establish clear team policies about what context can be shared with external AI services. Use local models or enterprise agreements for sensitive codebases.

**Risk 4 — Dependency confusion and hallucinated packages.** AI sometimes recommends npm/pip packages that don't exist, and attackers have begun squatting those package names with malicious code. Mitigation: always verify that any AI-recommended dependency actually exists and is maintained before installing it.

**Risk 5 — Overconfident security review.** Teams using AI to review security-sensitive code may develop a false sense of safety. AI misses classes of vulnerabilities that require understanding of runtime behavior, environment configuration, or cross-system trust relationships. Mitigation: AI security review is never a substitute for human penetration testing or a security audit.

**Risk 6 — Prompt injection in AI-assisted development tools.** Agentic AI tools that read your codebase can be influenced by malicious strings embedded in code or comments designed to redirect the agent's behavior. As agents gain more permissions (write access, command execution), this risk grows. Mitigation: audit what permissions your AI agents have, apply least-privilege principles, and review agent action logs.

---

**Q14. How do you think about intellectual property and licensing risks when using AI-generated code in production?**

**Sample Answer:**

This is legally unsettled territory, and anyone who tells you it's definitively resolved is wrong. That said, here's how I think about it:

**The core risk:** AI models are trained on code with various licenses, including GPL-licensed code. There's a reasonable concern that for some inputs, models may reproduce training data verbatim or near-verbatim, and if that code was GPL-licensed, you may be incorporating GPL code into a proprietary codebase.

**What we know:** GitHub Copilot has a documented feature where it can produce verbatim code matches from training data. The frequency and legal implications are debated. No major court has ruled definitively on whether AI-generated code can infringe copyright.

**My practical approach:**

First, I use tools that offer indemnification for IP claims — some enterprise offerings (GitHub Copilot Enterprise, certain cloud provider AI tools) offer contractual protection. This transfers legal risk to the vendor.

Second, I apply a "uniqueness check" for any substantial block of generated code — searching for distinctive segments to see if they appear in open-source repositories. Not perfect, but it surfaces the most obvious cases.

Third, for the most legally sensitive code (core product IP, code we'll license to others), I prefer to understand and rewrite AI suggestions rather than accept them directly. This creates a cleaner provenance story.

Fourth, I push for legal clarity at the organizational level. This isn't a decision individual engineers should make alone. Your legal team needs a policy on AI-assisted development, and that policy needs to address what AI tools are approved, for what use cases, and what the IP posture is.

The meta-point: the legal risk is probably lower than the paranoid view and higher than the dismissive view. Act proportionally and get institutional clarity.

---

## Section 7: Team Dynamics & Culture

---

**Q15. How do you onboard a team of experienced engineers onto AI-assisted development without either forcing adoption or losing the benefits?**

**Sample Answer:**

This is primarily a change management problem, not a technical one, and most tech leaders fail at it by treating it as technical.

**Principle 1 — Start with the pain points, not the tool.** Rather than "we're adopting Copilot," ask your team "what are the most tedious, time-consuming parts of your workflow?" Then demonstrate how AI addresses those specifically. Engineers who experience a direct benefit become champions. Engineers who are handed a mandate become resistors.

**Principle 2 — Separate skill levels.** Senior engineers who are deeply productive in their flow state may see net-negative initial impact from AI tools — the interruptions outweigh the value until they learn to work with it. Junior engineers often see immediate high-value gains on syntax and boilerplate. Forcing a uniform adoption curve ignores this.

**Principle 3 — Create explicit shared norms.** The most corrosive pattern is when half the team uses AI heavily and half doesn't, and this is never discussed. Code reviews become tense because different people have different implicit standards about AI-generated code. You need explicit team agreements: what do we review AI code for, what do we accept AI PRs without full review, what information do we not paste into external AI services.

**Principle 4 — Reserve spaces for AI-free learning.** Particularly for engineers earlier in their career, designate certain projects or problems as "work it out yourself" zones. AI tools are productivity amplifiers — they require a base of understanding to amplify. If engineers skip building that base, they're borrowing from a future they can't repay.

**Principle 5 — Measure outcomes, not usage.** Don't instrument Copilot acceptance rates as a performance metric. Instrument what you actually care about: defect rates, time to complete well-defined tasks, test coverage, system reliability. If AI adoption is improving these, the adoption is healthy. If it's not, it doesn't matter that everyone is using it.

---

**Q16. How do you calibrate how much to trust a junior engineer who produces code very quickly with heavy AI assistance?**

**Sample Answer:**

Calibration should happen at the *reasoning* level, not the output level.

The reason is: AI-assisted junior engineers can produce code that passes code review and tests but have no understanding of *why* it works, *what* its failure modes are, or *how* to modify it when requirements change. The code looks like a senior contribution; the understanding is not there. If you calibrate on output, you'll be wrong.

**My calibration approach:**

During code review, I ask questions that require *understanding*, not *recall*. "Why did you choose this data structure here?" "What happens if this service is unavailable when this function runs?" "How would you extend this if we needed to support multi-tenancy?" 

An engineer who used AI and understood it will answer these confidently. An engineer who used AI as a black box will be stuck. This distinction is crucial.

**Second test — watch them debug.** Debugging AI-generated code they don't understand is very hard. Give them a bug in their own code and watch the process. Engineers with genuine understanding navigate the codebase. Engineers who are AI-dependent tend to immediately paste things back into AI.

**Third test — whiteboard a related problem.** Not identical — related. Can they transfer their understanding to an adjacent problem? If they can, the AI assistance built on real understanding. If they can't, they have outputs without knowledge.

**The managerial obligation:** if a junior engineer is clearly AI-dependent without underlying understanding, the right response is not to ban AI. It's to create a deliberate development plan that builds the underlying competence — pairing sessions, code-free problem solving exercises, specific projects where they have to work without AI. This is a coaching problem, not a policy problem.

---

## Section 8: Advanced & Forward-Looking

---

**Q17. What do you think are the near-term limits of AI coding agents, and what would need to change for them to surpass those limits?**

**Sample Answer:**

The current limits cluster into four categories:

**Limit 1 — Context window and long-range coherence.** Even with large context windows, agents lose coherence over long task execution. They make early decisions that create constraints that make later decisions locally plausible but globally wrong. They'll implement a caching layer in one module and then implement an identical but incompatible one three modules later. What needs to change: better working memory architectures that maintain a *semantic* rather than just *token-level* understanding of system state.

**Limit 2 — Specification comprehension.** Agents are excellent at implementing what you say and bad at understanding what you mean. Requirements contain implicit constraints, organizational preferences, domain conventions, and non-obvious tradeoffs that aren't in the specification. An agent will implement a feature that satisfies the spec and violates three unstated conventions. What needs to change: better grounding in organizational and domain context, probably through long-lived memory and richer knowledge graph retrieval.

**Limit 3 — Uncertainty calibration.** Agents don't know what they don't know. When they hit a problem that requires information they don't have, they tend to make plausible-sounding assumptions rather than stopping to ask. Humans know when to escalate. Agents often don't. What needs to change: better uncertainty quantification and agent designs that are rewarded for appropriate escalation to humans.

**Limit 4 — Verification and self-correction.** Current agents can run tests and see if they pass, but they're bad at designing experiments to verify their own understanding. They can't say "I think this distributed system has a race condition but I can't reproduce it; let me design a stress test that would surface it if it exists." What needs to change: agents that can reason about their own blind spots and design verification strategies, not just execute specified tests.

**What would unlock the next level:** the combination of persistent, updatable world models for each codebase, better multi-step planning with explicit uncertainty tracking, and tighter integration of formal verification for specific claim types. This is a 3-7 year horizon for meaningful progress, in my view.

---

**Q18. How do you think about the long-term effect of AI coding tools on the skills required to be a great software engineer?**

**Sample Answer:**

This is where I'll give you a view that's probably more nuanced than the standard "AI is going to replace programmers" or "AI just handles the boring stuff, nothing changes" takes.

**What I think gets commoditized:** syntax fluency, library API memorization, boilerplate generation, and the ability to translate a well-understood algorithm into correct code. These have always been the *table stakes* of engineering, not the *value creation*. Losing the premium on these is not a tragedy.

**What I think gets more valuable:**

*Systems thinking* — the ability to understand how a complex system behaves under load, failure, and change. AI cannot substitute for this. It gets more valuable because AI raises the floor on execution, so the differentiator becomes design quality.

*Requirements analysis and problem decomposition* — translating ambiguous human needs into precisely-specified engineering problems. AI is a remarkably powerful answer machine to a well-formed question. The ability to *form the right question* becomes the scarce resource.

*Judgment under uncertainty* — knowing when to ship, when to refactor, when to rewrite, when to push back on a requirement. These are decisions that require integrating technical knowledge with organizational, product, and human context. This is not automatable.

*Debugging intuition* — the ability to form a hypothesis about a system's misbehavior from sparse signals. This requires a mental model of the system that AI cannot build for you. Engineers who deeply understand their systems will be dramatically more valuable than engineers who don't.

**The uncomfortable prediction:** I think we'll see a bifurcation. Engineers who use AI to *deepen* their understanding — who use it to explore ideas faster, test hypotheses more quickly, and read more code than they otherwise could — will become dramatically more capable. Engineers who use AI to *avoid* developing understanding will plateau early and find themselves unable to operate on complex systems. The distribution of engineering talent will become more bimodal, not flatter.

The skill that matters most for a 20-year career is not the ability to use today's AI tools well. It's the habit of relentless understanding — always asking why, always building mental models, always finding the limits of your own knowledge. That habit is what separates engineers who ride every wave of new tooling from engineers who get stranded by it.

---

**Q19. If you were building an internal AI-assisted development platform for a 500-person engineering org, what would you build and why?**

**Sample Answer:**

Most organizations are solving this wrong — they're buying point tools (Copilot here, AI review there) and calling it a strategy. I'd build a unified platform around a central insight: **the value of AI assistance is proportional to the quality of organizational context it has access to.**

**Layer 1 — Knowledge graph construction.** Ingest the codebase, ADRs, incident reports, runbooks, Slack discussions, and Jira tickets into a continuously updated knowledge graph. This becomes the organizational memory that all AI tooling queries. The goal is that when any engineer asks an AI tool a question, it answers *in the context of this organization's specific systems, decisions, and conventions* — not generic internet knowledge.

**Layer 2 — Context-aware code assistance.** An IDE integration that, when you're working in a module, automatically retrieves relevant ADRs, related incidents, the engineers who best understand this code (from git history), and similar patterns from elsewhere in the codebase. Inline completion becomes dramatically more relevant because it has real organizational context.

**Layer 3 — Institutional review assistant.** Code review tooling that doesn't just look at the diff but understands: what systems does this change affect, what's the incident history of those systems, what conventions apply to this part of the codebase, and what do our internal experts say about this class of change.

**Layer 4 — Onboarding acceleration.** New engineers interacting with a codebase through an AI that has organizational memory can get up to speed dramatically faster. "Why is this service designed this way?" should surface the ADR and the Slack thread from 18 months ago, not require hunting down the one person who knows.

**Layer 5 — Observability over AI assistance.** Instrument everything. Which AI suggestions are accepted vs. modified vs. rejected? What are the defect rates of AI-generated code vs. human-written code? What are the patterns in AI review that correlate with post-merge bugs? Use this data to continuously improve the system and have honest conversations about where AI assistance is and isn't delivering value.

**What I'd explicitly not build:** a tool that tries to replace human judgment on architecture, security, or product decisions. The platform exists to make human engineers dramatically more capable. The moment you try to automate the judgment layer, you've built a liability, not an asset.

---

These 19 questions span the range from first-principles thinking to practical tooling to forward-looking vision. The answers I'd be most impressed by are the ones that show **genuine calibration** — knowing where AI helps, where it fails, where it creates risk, and how to design systems and cultures that get value from it without being naive about its limits.