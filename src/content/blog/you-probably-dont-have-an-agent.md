---
title: "You probably don't have an agent"
description: "Why most production GenAI systems are workflows wearing agent costumes — and why naming that distinction correctly matters more than it sounds."
pubDate: '2026-06-29'
readingTime: '9 min read'
---

There's a specific kind of evening you spend after you've built an agent for something that didn't need to be one. You're three coffee cups in, staring at a graph executor trace that says the node fired but the output is wrong, and you slowly realize the problem isn't the bug you're chasing. The problem is that you spent six weeks building a graph to solve what was always a pipeline.

I've had that evening more times than I'd like to admit. The more I see it in other engineers' systems too, the more I think it's the most common failure mode in production GenAI right now. It isn't bad prompting. It isn't poor retrieval. It isn't even hallucinations. It's that we keep building agents for problems that don't need agents.

So when someone asks me which agent framework they should use for their GenAI project, I usually answer with a question. Are you sure you have an agent? Most of the time, they aren't. They have a workflow with an LLM in it. And I think most of us, myself included until recently, have been talking around that distinction instead of about it.

## Most production GenAI failures are naming failures

Anthropic made the agent-versus-workflow distinction publicly in their "Building Effective Agents" post in late 2024. Others have echoed it since. I'm going to make it again here, sharper, and add what I think most of those takes have been missing.

The deeper claim is this: most production GenAI failures aren't model failures. They're naming failures.

You called the thing an agent. So you reached for an agent framework. So you built it like an agent. So when it broke, you debugged it like an agent. The label silently chose your architecture, your tooling, your mental model, and the surface area of your bugs. And you never noticed because the label felt natural. Everyone else was calling their systems agents too.

This is why getting the name right early matters more than it sounds like it should. The word you use for your system in week one determines what you build in week three, which determines what you can debug in week eight, which determines whether the thing ships at all in week twelve. Naming a system is a design decision, not a description of one.

## Agency is a property, not a vibe

The word "agent" has been doing too much work in this industry. It's been stretched to cover anything that calls a model, anything that uses tools, anything that has more than one prompt in a row. That's not a definition. That's a vibe.

Here's the engineering definition I find most useful: **a system is agentic if model output changes the program's control flow at runtime.**

That's it. That's the whole test.

This isn't the only definition of an agent. Ask ten researchers and you'll probably get eleven answers. It's simply the one that gives me a concrete engineering test instead of a philosophical debate.

If your code is `step1 → step2 → step3 → done` and the model just produces text inside each step, you do not have an agent. You have a pipeline. The fact that the steps involve LLMs is incidental. The fact that you imported LangGraph, CrewAI, or AutoGen to model it doesn't make it agentic. Cosplay is not architecture.

If your code actually branches, loops, or hands off based on what a model said, then you have agency. The model is steering the program. That's the thing.

## The diagnostic question

Before you pick a framework, before you sketch a graph, before you reach for the LangGraph or CrewAI quickstart, ask this:

> If I imagine running this system a thousand times on a thousand different inputs, do the steps it takes change based on what the model decides? Or do the same steps run every time, in the same order, with only the contents of the steps differing?

If it's the second one, you have a workflow. Build it like a workflow.

I've started calling this control flow honesty. Most engineers I know default to flattering descriptions of their systems: supervisor patterns, planning agents, multi-step reasoners. The honest version is usually less impressive. And the less impressive version is almost always the one that ships.

![Workflow vs agent diagram](/diagrams/workflow-vs-agent.svg)

## A real example

Let me make this concrete with something I actually built. I'm going to keep it sanitized because the production system lives at my employer, but the shape is exactly right.

The system is a technical service assistant. Internal users ask it questions about our products. It pulls relevant context from several sources of varying authority, official product specs on one end, casual ticket resolutions from years ago on the other, synthesizes one grounded answer, and checks that answer against the spec before returning it.

When I first sat down to build this, my default reach was for a graph-based agent framework. That's what I'd been using for everything for about a year, and once you have a hammer that fits your hand, you stop checking if the thing in front of you is actually a nail.

I drew a graph on the whiteboard. I named the nodes. I drew the edges. I started sketching state objects. And about halfway through that exercise I noticed something uncomfortable: the graph had no decision points. There was nothing in it for the model to decide that affected which node ran next. The edges weren't conditional. The control flow was a straight line dressed up in graph notation.

I had been drawing a flowchart and calling it a graph.

The actual steps, written honestly, were:

1. Take the question and resolve which product it's about.
2. Pull from source A, a structured store.
3. Pull from source B, a document store.
4. Reconcile the two using a known authority hierarchy.
5. Synthesize an answer.
6. Run a consistency judge against the spec.
7. Return.

Apply the diagnostic question. Across a thousand different incoming questions, does the sequence of steps change? No. Step 1 always runs. Step 2 always runs. The judge always runs. The model isn't deciding which sources to consult or in what order. It's just producing text inside fixed boxes.

That's a workflow. A workflow with several language model calls in it, but a workflow all the same. Calling it an agent would have been flattering it.

## What you lose when you treat a workflow like an agent

You might be reading this and thinking, fine, it's a workflow, but who cares? If LangGraph or CrewAI can model it, why not just use them? The framework hides the complexity and I can move on with my life.

Here's what you actually lose.

**You optimize the wrong layer.** Agent frameworks are optimized for routing, branching, and state management. Those capabilities are valuable, but if your control flow is fixed they usually aren't the layer that determines whether your system succeeds. In my experience, production quality comes down far more often to retrieval quality, prompt design, evaluation, and how information flows between LLM calls than to how the workflow is orchestrated.

**Debugging goes from boring to mystical.** When something breaks in a linear pipeline, you read the function top to bottom and find it. When the same logic is wrapped inside a graph executor, you're now debugging both your code and the framework's interpretation of your code. I've spent evenings I'd rather not get back tracing why a node "wasn't firing" when the real answer was that I'd modeled a for-loop as a graph for no reason at all.

**You hide the actual problem from yourself.** When the abstraction is "agent," you stop asking sharp questions like "what's my source authority hierarchy and where is it encoded?" and start asking soft ones like "should this be a supervisor pattern with a planner node?" The first question is real engineering. The second is decoration.

And here's the part nobody talks about: the cost isn't just engineer hours. It's organizational credibility. When a system breaks in production and the team can't explain why, because the architecture is more complicated than the problem it's solving, stakeholders don't lose faith in the team. They lose faith in GenAI. Every over-engineered "agent" that misfires in production makes it incrementally harder for the next legitimate GenAI project at the company to get budget, headcount, and political air cover. Complexity is a debt you should take on deliberately, not by default. And the interest gets paid by people you've never met, on projects that haven't started yet.

## Framework features can become framework gravity

There's a fair pushback to all of this. People don't reach for these frameworks only because they need branching. They reach for them because they want checkpointing, durable execution, tracing, observability, human-in-the-loop, or streaming. Those are real engineering problems, and good frameworks do solve them well. I'm not picking on any one of them. LangGraph, CrewAI, AutoGen, the OpenAI SDK's agent abstractions, and whatever ships next quarter all sit in roughly the same shape. The argument is about the category, not the roster.

The mistake isn't using those features. It's assuming they require an agent architecture. Every framework you adopt brings its own baggage: an execution model, a state abstraction, a debugging surface, a way of thinking that quietly shapes how you write code. If you actually need that baggage, the framework pays for itself. If you don't, you're carrying weight for no reason. In many workflow-heavy systems, a combination of ordinary Python and purpose-built tooling for tracing, evaluation, and observability is enough. You don't need to couple those operational concerns to your execution model. Use a framework because you need its capabilities, not because calling your workflow an "agent" makes the architecture feel more sophisticated.

## What to do instead

Once you've named the thing honestly, pick tools that solve the bottleneck you actually have. Start with the simplest architecture that correctly models your system, and add abstraction only when the problem demands it.

If the hard part is prompt quality, signatures, and systematic evaluation across multiple LLM calls, DSPy is an excellent fit. It treats prompts as programs you can compile and optimize against an eval set. That's exactly the problem I'm usually trying to solve.

If the workflow is relatively straightforward, ordinary Python is still an underrated orchestration framework. Pair it with MLflow for experiment tracking, tracing, evaluation, and observability, and you'll satisfy many of the operational requirements that lead teams to adopt larger orchestration frameworks while keeping the execution model obvious. The most underrated framework in 2026 is still `def main():`.

And if your system genuinely needs runtime planning, dynamic routing, checkpointing, resumability, or long-lived state, then yes, reach for a framework built for that. LangGraph, CrewAI, AutoGen, and others all exist because those are real problems worth solving. The point isn't to avoid these tools. The point is to make sure the problem you're solving actually needs them.

## The one line to remember

If you take nothing else from this post, take this:

**The first question in a GenAI project isn't which framework. It's whether you need one.**

Most production GenAI work I've seen, including most of my own, is workflows wearing agent costumes. The industry has been shipping graph libraries to people who needed a for-loop and a good prompt. Architecture should follow behavior, not marketing.

You probably don't have an agent. That's not a failure mode. That's the normal case. Simple architectures, built honestly to fit the problem, do most of the real work in production. They don't make for impressive diagrams. They just work.

And here's the part I wish someone had told me a year ago. The engineers I've watched go furthest in this field aren't the ones who reach fastest for the trendiest framework. They're the ones who can look at a system and tell you, in one sentence, what control flow it actually has. That skill compounds. The frameworks you'll be using in three years probably don't exist yet.
