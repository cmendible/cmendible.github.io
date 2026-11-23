---
author: Carlos Mendible
categories:
- azure
- ai
- management
date: "2026-11-23"
description: 'What marathon running taught me about operating AI agents: intelligence is becoming a commodity, operational discipline is not.'
draft: false
tags: ["agents"]
title: 'The AI Agent Advantage Is Endurance, Not Intelligence'
---

> *What marathon running taught me about operating AI agents*

**Key takeaways**

1. **Intelligence is becoming a commodity. Operations are not.** Strong models will soon be everywhere. What's rare is running agents reliably, securely and profitably month after month.
1. **Trust is a performance issue, not only a compliance one.** If people don't trust an agent, they won't use it, leaders won't scale it and security won't approve it. Build security in before launch.
1. **Measure value, not impressions.** Base decisions on evidence about the agent's quality and the value it delivers. Treat observability as the one shared record everyone works from.
1. **Plan a rest week before launching more.** Most organizations are overtraining, shipping new agents without consolidating the ones they have. For every agent in production, name its owner, how often a person has to fix its work, and what each successful outcome costs.

When I started running marathons, I thought improvement was about effort. More kilometers, more sessions, push harder when it hurts. More work, better results.

Marathons taught me otherwise.

The runners who consistently hit their goals were rarely the ones training hardest. They were the ones listening most carefully. They tracked recovery as closely as pace, measured progress instead of guessing, and watched long-term trends rather than obsessing over a single workout. Performance wasn't really about running. It was about managing a system. Race day was simply the visible result of hundreds of small decisions made over months: push or recover, trust emotion or data, treat the symptom or fix the cause.

Lately, I've found myself returning to those lessons while working with AI agents. One lesson in particular, recovery, is the one I think the industry is missing most.

## Intelligence is becoming a commodity. Operations are not.

The AI industry is obsessed with intelligence. Every week brings a new model, a new benchmark, a new prediction about autonomy. Much of that progress is genuinely important.

But I believe we're focused on the wrong race.

Many organizations no longer have an AI capability problem. They have an AI operational maturity problem. They can build impressive agents; what they struggle with is running them reliably, safely, and profitably, month after month. BCG's 2025 study of more than 1,250 companies found that only 5% achieve AI value at scale, and 60% see little to no value. The gap? Over 60% of the leaders rigorously track AI value, versus only 17% of the stagnating companies.

Models will keep differing at the margins, and for some specialized tasks the choice of model will still matter. But enterprise technology follows a familiar pattern: infrastructure becomes accessible, capabilities spread, features become commodities. Within the next few years, most organizations will have access to highly capable models. What stays hard is operating complex systems consistently, securely, and at scale, and that is where competitive advantage will come from. The winners won't be the organizations that train hardest. They'll be the ones that recover best: the ones that pause to measure, fix, and consolidate while everyone else keeps shipping.

## We've solved this kind of problem before

The technology is new. Many of the operational challenges are not.

A marathon runner is a complex, adaptive system shaped by training, sleep, nutrition, stress, weather, injuries, and race strategy. No coach can predict race-day performance precisely. Yet elite athletes deliver remarkable results consistently, not by eliminating uncertainty but by managing it. They measure what matters, monitor continuously, evaluate objectively, and spot risk early.

Many AI initiatives do the opposite. They pour effort into making agents more capable and comparatively little into the operational systems that keep them performing. A great demo creates excitement. A successful launch creates optimism. Neither guarantees value. That comes when an agent is still performing six months later, after requirements have shifted and data has drifted.

Four disciplines from endurance sport map almost directly onto running agents in production. Together they are what make recovery possible, because you can't rest what you can't see, measure, or trust.

## 1. Security is the pre-season medical check

Before serious marathon training, you want to know your body is ready. Medical checks exist because finding a problem before the race beats finding it at kilometer 30. Their purpose isn't to slow athletes down; it's to let them perform safely.

In sport this is obvious. In technology, it somehow still feels optional.

Before an agent touches corporate knowledge, customer data, or operational and financial systems, organizations should be able to answer a few basic questions:

- What data can the agent access, and what must it never access?
- Which actions can it take on its own, and which require human approval?
- What happens if its instructions are manipulated through prompt injection?
- How would we detect and contain the exposure of confidential information?

Gartner predicts that over 40% of agentic AI projects will be canceled by the end of 2027, pointing to rising costs, fuzzy business value, and weak risk controls. These are usually framed as governance questions. I'd argue they're performance questions, because adoption depends on trust. Users won't rely on systems they don't trust, leaders won't scale them, and security teams won't approve them. Trust can't be bolted on after deployment. It has to be engineered in from the start.

## 2. Evaluations are the training log

Serious runners love metrics, sometimes too much: pace, cadence, heart rate, lactate threshold, VO₂ max, training load. What you can measure, you can improve.

Marathon preparation taught me that feelings are unreliable. I had sessions where I felt terrible and performed exceptionally, and others where I felt strong and the data said otherwise. The numbers weren't perfect, but they were usually more honest than I was.

AI agents need that same honesty. Too many organizations still judge success with phrases like "the outputs look good," "users seem happy," or "we haven't had major complaints." Those signals matter, but they aren't evidence. Once agents sit inside critical processes, teams need objective answers:

- How often is the task completed correctly?
- How often does a human have to step in?
- Which scenarios consistently fail?
- Did the last prompt, model, or tool change improve things, or quietly introduce a regression?

But technical metrics are only half the story. You don't want to operate on intuition, and you don't want to operate on technical metrics alone. A 95% task completion rate says nothing about whether the business is better off. What you want is a clear, measurable ROI: what the agent costs, what it saves or earns, and whether the business outcome is actually moving. That's how you decide which agents deserve more investment and which should be fixed or retired.

## 3. Observability is the heart rate monitor

If evaluations tell you *whether* an agent is performing, observability tells you *why*.

A heart rate monitor doesn't tell you the finish time; it tells you what's happening inside you while you run. A rising heart rate at the same pace can be the early warning that something is off.

Agents need the same inner view. Traditional monitoring was built for predictable systems, where you knew what failure looked like. Agents are probabilistic and can fail with plausible but wrong answers, drift or runaway costs. Yet many organizations still see only the prompt and the response, the pace without the heart rate. Teams need one end-to-end trace across the agent, its tools, the model and the infrastructure. It should show which tools were called, which sources shaped the answer, where time and tokens went, and why an action failed.

This isn't just debugging. You can't govern what you can't see, and you can only improve what you understand. We learned this with cloud computing, when systems became too complex to run blindly. Agents are on the same path, only faster.

## 4. Every great athlete has a coach

Deep into a hard training block, your ability to judge your own situation degrades. That's why coaches matter. They see what athletes miss, challenge assumptions, and improve decisions.

Agents need a similar relationship with people. Autonomy is not the goal; better outcomes are. Humans bring context, judgment, ethics, and business understanding. Agents bring speed, consistency, scale, and pattern recognition. The strongest organizations won't remove humans from the loop. They'll design it so people and agents make each other better, and they'll expand autonomy gradually, as evidence earns it.

A practical test: for each agent in production, can you name its owner, its review cadence, and the threshold at which a human steps in? If not, it's running on hope.

## The industry has an overtraining problem

The most valuable lesson marathons taught me wasn't about running. It was about recovery.

Fitness isn't built during the workout. It's built afterward, when the body adapts. Recovery isn't lost time; it's part of training. Skip it, and performance stalls, then declines.

Looking at the AI landscape, I'm sure many organizations are overtraining. A new model, framework, or architectural pattern arrives every week, and teams, leaders, and vendors all feel the pressure to keep going.

I often ask organizations a simple question: do you know which of your AI investments are actually creating value? Surprisingly often, the answer is unclear. Deloitte's 2026 survey of 3,235 leaders found that only 25% have moved 40% or more of their AI pilots into production. In my experience, the cause is rarely the technology and more often operational maturity that can't keep pace: governance lags. Observability becomes an afterthought. Evaluation frameworks never mature. Technical debt piles up.

That's the organizational equivalent of running every session at race pace. Eventually something breaks. Sustainable progress needs deliberate periods of consolidation: time to standardize, measure, and harden what's already deployed before adding more. Think of it as a rest week in a training plan: scheduled, recurring, and with a clear exit criterion, such as every production agent having an evaluation suite and a known cost per successful outcome.

## Deployment isn't race day

Non-professional marathon runners prepare for months to race once. An agent runs the race every day, sometimes several times a day, against changing data, shifting requirements, and rising expectations.

That changes what deployment means. It isn't the finish line, or even the starting gun. It's the first day of a training season that never ends. Every discipline that prepares a runner (the medical check, the training log, the heart rate monitor, the coach, the rest days) has to keep running for as long as the agent does.

When I started running, I thought results came from effort. Today, many organizations think AI results come from intelligence. Both miss the same thing. Effort and intelligence are the engine. The system around them decides whether that becomes sustained performance.

My prediction is simple. As models improve, intelligence is increasingly becoming available to everyone. Operational discipline isn't. The organizations that win with AI won't be the ones with the most agents or the latest models. They'll be the ones with an operating model that secures, governs, evaluates, observes, and continuously improves what they run, and ties all of it to business results. That's what lets AI move from pilot to production, and from production to return. AI doesn't need a new playbook, just the discipline endurance sport has spent decades refining.

> Before you launch your next agent, take a rest week for one you already run. Ask who owns it, how often a person has to fix its work, and whether each successful outcome is worth what it costs. The answers will tell you more about your AI strategy than the next model release.

In marathons and in AI alike, peak performance gets the attention. Sustainable performance wins the race.

---

**Sources**

- BCG, [The Widening AI Value Gap: Build for the Future 2025](https://www.bcg.com/assets/2025/the-widening-ai-value-gap.pdf), September 2025.
- Gartner, [Gartner Predicts Over 40% of Agentic AI Projects Will Be Canceled by End of 2027](https://www.gartner.com/en/newsroom/press-releases/2025-06-25-gartner-predicts-over-40-percent-of-agentic-ai-projects-will-be-canceled-by-end-of-2027), June 2025.
- Deloitte, [State of AI in the Enterprise 2026](https://www.deloitte.com/us/en/about/press-room/state-of-ai-report-2026.html), January 2026.
