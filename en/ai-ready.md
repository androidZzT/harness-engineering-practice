# On AI-Ready and AI-SDLC

> A standalone piece on an adjacent topic — not a numbered chapter of the Harness Engineering series. The harness is about "the runtime layer around the agent"; this piece is about something more upstream: is your engineering itself ready for AI to get its hands on?

Lately I've seen plenty of teams dive straight into a "multi-agent collaboration platform": building an orchestration framework, designing role splits, tuning a workflow engine, itching to have a swarm of agents carry a requirement all the way from intake to release. But in actual use, the AI in their codebase is the same as ever — it makes the mistakes it always made and skips the steps it always skipped.

The problem is usually not that the agent isn't strong enough. It's that the engineering itself isn't ready for AI.

That brings up an underrated prerequisite: AI-Ready. Before you've reached it, rushing to grind on an Agents platform is mostly putting the cart before the horse.

## 1. First, separate two terms: AI-SDLC and AI-Ready

Start by pulling apart two concepts that are easy to confuse.

AI-SDLC is the AI-assisted software development lifecycle. From requirement clarification and solution design through coding, integration, testing, and release, every stage has AI deeply involved — the goal being to let agents do as much as possible and humans step in as little as possible, while keeping delivery quality.

AI-Ready is a different thing: whether your code repository, requirement documents, and development process are actually ready for AI-SDLC. The phrase comes out of engineering practice at companies like Amazon. Its core isn't chasing a "perfect 100-point ideal state," but identifying and clearing out the factors that are "definitely unfriendly to AI."

A loose analogy. AI-SDLC is "have the AI do the work"; AI-Ready is "tidy the site first so the AI can get its hands on it." One is the action, the other is the prerequisite.

It has another property: you can't declare yourself Ready by eyeballing a score. Whether a repository is Ready or not only gets verified inside real requirement delivery. It's a posterior, continuously improving state — not a one-time certificate.

## 2. Grinding on an Agents platform before you're Ready is backwards

A lot of investment today goes into "stronger agent orchestration": how multiple agents divide work, how they pass context, how the workflow is orchestrated. These have value, of course — but a prerequisite often gets skipped: the repository the agent works in has to be one the AI can actually run in.

However smart the agent is, drop it into a repository that won't even start locally, has no coding standard, is packed with dead code, and keeps its requirements in a wiki — and it's stuck all the same. Either it can't run, or it converges painfully slowly, trial-and-erroring its way.

Switch the scene and it's obvious: you hired a great chef, but the kitchen has no power, the ingredients aren't washed, and the recipe is scrawled across a pile of sticky notes. The meal turns out badly — and it isn't the chef's fault.

So the relationship between AI-Ready and AI-SDLC is this: the former is the latter's prerequisite and direction of reform. Whether the repository is cleanly structured, whether technical debt is under control, whether requirements are structured enough for the AI to understand, whether CI can let the AI run on its own — leave these unsolved, and any fancy agent collaboration built on top is built on quicksand.

Get the order wrong and the cost is concrete: you're spending expensive agent compute to repeatedly slam into walls of technical debt that a little effort could have torn down.

One number captures the gap. In some teams' practice, after putting a single-domain repository through one serious round of AI-Ready work, the AI's code acceptance rate rose from the low 60s to above 90 percent. The model didn't change; only the environment it faced did. The same agent produces wildly different output in a tidied repository versus a messy one.

## 3. What AI-Ready actually requires

AI-Ready isn't mystical. On the ground it's two things: build a knowledge base, and flesh out the skills for each delivery stage.

First, a useful method. Within a team, it's hard to reach consensus on "what is AI-Ready," but easy to agree on "what is *not* AI-Ready." So don't chase perfection up front — drive by problems, and clear the obvious blockers one by one.

Walk through it by delivery stage; the typical "not AI-Ready" looks like this:

- Requirement stage: requirements live in a wiki or get passed along verbally, instead of structured markdown; no product SPEC knowledge base, so the AI can't grasp the business semantics.
- Solution stage: no architecture docs, so the AI can't locate where to change things; the core architectural constraints exist only in a few people's heads.
- Coding stage: the repository has no coding standard; it's piled with dead code; naming is arbitrary, comments are absent; God Classes everywhere.
- Testing stage: the service won't start locally; the core path has no automated cases; nobody ever documented how to construct test data.

Those are the anti-patterns to fix first.

The heavy part of the work is the knowledge base — it's the foundation of AI-Ready. A fairly mature approach is three layers: general knowledge, domain knowledge, requirement-level knowledge, narrowing toward the specific layer by layer.

What goes into the knowledge base? Architecture design, foundational capabilities, the foundational component library — these are the AI's footing for understanding a system. Unlike a veteran employee, the AI has no built-in picture of "roughly what this system looks like, how this kind of thing is usually done"; you have to write that down explicitly and feed it.

Two counterintuitive but important realizations here.

One is "code as knowledge." The only thing that accurately reflects how the system actually runs is the code itself; a wiki guarantees neither completeness nor agreement with the current state. So knowledge retrieval should center on code the AI can read, with the wiki only filling in the stable fragments that aren't visible from the code.

Two is the priority of knowledge. In testing, the various kinds of knowledge don't help the AI equally: best-practice knowledge helps the most, architecture knowledge next, standards and terminology trail behind. And one very concrete finding — semantic naming contributes the most, because it directly determines whether the AI can find the right place to change. Giving a variable or function a name that states its intent is sometimes worth more than a pile of docs.

The other thing is fleshing out the skills for each stage. Distill the things "only an old hand knows how to do" into capabilities the AI can call directly: assessing a repository's Ready level, landing coding standards, doing interface-level unit tests, generating structured requirements. Pair every delivery stage with its skill, and the AI knows how to act at that step and what bar to hold.

## 4. After you're Ready, then talk about workflow and agent collaboration

Lay the foundation first — knowledge base, per-stage skills, a service that starts, tests that pass — and only then is it worth optimizing workflow interaction and multi-agent collaboration.

This step enters continuous evolution, shifting from "clearing blockers" to "metric-driven."

Concretely: collect process data across the stages of AI-SDLC, and watch the negative indicators in particular — how many times the AI's output got flagged as wrong, how many rounds of re-clarification were needed, how often a human had to step in, what fraction was wasted execution. These indicators are far more reliable than "the AI feels pretty good"; they point straight at where it's still stuck.

Then drive improvement backward from the negative indicators: is the knowledge base missing content, does the harness need tuning, is the spec template incomplete, or does some skill need iterating. Verify after each change, forming a loop of "observe data → identify problem → targeted optimization → verify effect," and let the Ready level climb a little at a time.

Order matters here too. Get a single agent running smoothly in a clean environment first, then talk about how multiple agents divide and collaborate. Do it the other way and you're optimizing a collaboration process built on quicksand — the more you tune it, the worse the rework later.

## Closing

Back to the misconception we opened with. Multi-agent collaboration is a good story, and platforms and frameworks are genuinely useful. But before you pour resources in, it's worth asking yourself one thing: is my engineering AI-Ready?

AI-Ready isn't sexy. It doesn't pitch as well as "a swarm of agents finishes the work on its own." Yet it's exactly the unglamorous work — straightening out naming, clearing dead code, distilling architecture and components into a knowledge base, filling in each stage's skills — that decides whether the agent layer on top can run at all.

Stack things up before the foundation is solid, and the cost of later rework runs much higher than doing the honest reform from the start.
