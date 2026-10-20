# [Development] Expert Intuition Is the Ability We Need Most in the AI Era

[English](./01-Development-AI-Era-Expert-Intuition.md) | [繁體中文](./01-Development-AI-時代最需要的能力是專家直覺.md)

I think people who use AI will feel this especially keenly. AI can finish a great deal of repetitive work more efficiently. It can even reason and make guesses to some degree, helping people with certain tasks involving “creation and imitation.” This makes “producing results quickly” less important than it used to be, while increasing the importance of the soft skills needed to put work into practice in a business, or to fully “move the current state toward the desired outcome.”

I believe the foundation of those soft skills is “**expert intuition”**. AI’s output ultimately enters the human world: **responsibility for that output**, **willingness to buy**, and **judgments about its use all come from people**. For now, AI is an accelerator for the production process. By speeding that process up, it makes the soft skills needed to carry work through to completion more visible. I will use software engineering as my example below; readers in other industries can draw the corresponding parallels.


### What Is **Expert Intuition**?

Readers unfamiliar with **Expert Intuition** can consult these academic definitions

1. [Conditions for intuitive expertise: A failure to disagree.](https://psycnet.apa.org/record/2009-13007-001)
2. [Dan Kahneman and Gary Klein on strategic decisions: When can you trust your gut?](https://bobmorris.biz/dan-kahneman-and-gary-klein-on-strategic-decisions-when-can-you-trust-your-gut)

This article will not pursue an academic discussion of the term. It will offer an initial account of **“expert intuition when software engineering enters a business setting.”** First, look at this image

![Expert intuition in the AI spotlight](../assets/agent-thoughts/ai-software-development/01-Expert-Intuition-in-the-AI-Spotlight.webp)

AI Expert Intuition

To complete a software development project and put it to work in a business setting, Coding is only part of the job (the area illuminated by the flashlight in the image). The full job also calls for **architecture design, error diagnosis, trade-offs, upstream and downstream integration, release criteria, version control, and conversion analysis (the areas outside the light in the image)**, as well as the long-term evolution, collaboration, Schedule, strategy, evaluation, and profitability that business needs demand. That does not even cover an understanding of a business’s gray areas and unwritten rules. AI cannot currently do all this, yet this is precisely where software creates value in the human world.

### What I Mean by “Expert Intuition”

In **software engineering**, expert intuition means the ability to achieve “complete business implementation.” I break it down as follows:

> ***Expert Intuition in Software Engineering = Architecture Design + Business Experience***

![Expert intuition connects a full range of capabilities](../assets/agent-thoughts/ai-software-development/02-Expert-Intuition-Connecting-a-Full-Range-of-Capabilities.webp)

#### **Architecture Design**

**Real-world development tasks have no standard answers**. Software development must adapt to user numbers, target groups, time pressure, international markets, regions, beliefs, cultures, and more. A true expert must therefore be able to do the following

1. **Trade-offs**: Explain why an architecture “chooses A instead of B, C, D, or E,” and whether it follows suitable design patterns, rather than coming back six months later to find that people can barely maintain it and are trapped in the development hell of ACA (AI Check AI).
2. **Error diagnosis**: When AI presents an interpretation that sounds plausible but is wrong, produces tightly coupled and redundant Code, or causes any other failure that stops the software from running, you must **spot and resolve the problem**. The classic question is, “It runs normally on my phone, so why doesn’t it run on anyone else’s?” In practice, you cannot gather every user’s phone and have AI test each one. If the problem is in production, the time to fix it is even tighter: every extra minute of failure means more losses. AI will not be responsible for those losses; you will.
3. **Problem boundaries:** At this stage, if you do not mention or ask about something, AI will not answer by default unless the point is highly relevant and makes it into the Top K of the Retriever’s analysis. Real bugs, however, often arise in rare situations. For example: in a WeChat Miniapp, the Request Head length must be <= 1024 Byte. Once the Head grows past that limit, the result is Request Fail. Someone unaware of this boundary who develops and releases a product using only AI could cause a disaster.
4. **Problem definition:** Real development often requires Performance improvements, but which target User group should you focus on? Which metric? Should you target Users with older devices or another group? How do Performance metrics such as TTFT / LCP / QPS relate to business metrics such as DAU and MAU? Where should you start to improve conversion rates? AI still cannot connect these things naturally. Even if it gives you numbers, they may be limited estimates or arbitrary guesses.

#### **Business Experience**

When software enters a business domain and becomes a development task, the work still centers on real situations, especially upstream and downstream integration, release timing, coordination with campaigns, and past disasters

1. **Upstream and downstream integration**: A mature, large application is not built by a single team. Completing one process often depends on dozens, or even hundreds, of Key / Value pairs. You must identify which Value the feature under development should use and which Value, if changed, would leave your own team unaffected but create problems for upstream or downstream teams. Each team controls access to its Code, so an external AI cannot read it at will. Even if another team gives you access, what about its upstream and downstream teams?
2. **Release timing:** In what real-world setting will this requirement be released? At the simplest level, could you release it for Christmas or Black Friday and use people’s expected buying behavior to increase business revenue?
3. **Past disasters:** Avoid changes like those that caused disasters before. For example: when a campaign suddenly brings in many more users, the APP must not crash, the Server must not become overloaded, and duplicate actions or orders must not appear. An open-source provider might also change something and cause a disaster for downstream teams using its Tool. One example is [the Ant-design Christmas Easter egg disaster](https://github.com/ant-design/ant-design/issues/13848).

### Advancing Expert Intuition**:** The Ability to Move the Current State Toward the Desired Outcome

I break down the soft skills that are truly scarce in the AI era one step further:

> ***The Ability to Move the Current State Toward the Desired Outcome = Expert Intuition + Leadership Without Authority***

![Roadmap from expert intuition to business impact](../assets/agent-thoughts/ai-software-development/03-Roadmap-from-Expert-Intuition-to-Business-Impact.webp)

#### **Leadership Without Authority**

Collaboration at work or in open-source communities requires a great deal of communication, and you will not always have direct authority over others. **Discussing solutions** with people outside your team and **increasing their willingness to act** are real parts of the work that AI cannot do for you

1. **The willingness of collaborating teams:** In the AI era, people’s time and attention are scarce resources. Upstream and downstream teams have their own requirements, Priorities, and task Schedules. Even if AI can produce Code quickly, you still need those teams to make time to talk with you. It matters that you get your requirement onto their Schedule and raise its Priority so they are more willing to carry it out and cooperate, because AI cannot persuade your colleagues for you.
2. **Clarifying dependency costs:** In the multi-team setting described above, completing your task may require a dependency path, for example: your task ➔ Team A ➔ Team B ➔ Team C. A delay or error at any point will prevent the task from being completed on time. What matters is whether you can agree on the deliverables and contracts across teams, make sure everyone knows their role in the process, and show how much a delay would affect the work. AI cannot complete this task for you either.

### Conclusion: Expert Intuition Matters More in the AI Era Than Before

In an era of explosive AI growth, everyone asks how to use fewer Tokens, lower costs, and reduce the time people spend on work. But we should sometimes stop and ask whether, **without using AI, burning Tokens, or wasting time waiting for AI to respond**, you can already **recognize problems in real business settings and propose solutions quickly**. Otherwise, you may spend all those resources only to discover that **a problem was there all along, AI led you in the wrong direction, or outside dependencies will prevent you from finishing on time**.

I wish every professional these three abilities:

1. A basic ability to use AI precisely
2. Expert intuition to judge whether AI’s output is correct
3. The soft skills to move the current state toward the desired outcome

These are the scarcest abilities in the AI era. They prevent us from using AI to produce a superficial result quickly, only to find that it does not meet real requirements, that people cannot use it with confidence, and that, if something goes wrong, people cannot find a starting point for fixing it. In the end, the work becomes a vicious cycle of ACA (AI Check AI).
