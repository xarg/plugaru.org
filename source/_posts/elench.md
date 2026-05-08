---
title: Elench 
date: 2026-05-08 10:54:06
categories: startups
tags: ai, startups, manufacturing, robotics 
---

### I'm leaving SaaS to make things in the physical world

After 11 years at [Gorgias](https://gorgias.com) I've stepped down from my CTO role and started [Elench](https://elench.com). The mission is the same: automate all the things! This time around it's more about atoms than bits.

Fully automated manufacturing is not a new idea by any measure, but I think now is the right time due to a confluence of factors below:

### Increased demand

As I write this, I consider it self-evident that the US and EU are trying to reindustrialize fast and it doesn't appear to depend on who's in the White House or in Brussels. In the past 30+ years we've been outsourcing everything that's not super critical to Asia and we didn't capture the learning curve of consumer manufacturing, which makes it even harder to reindustrialize. The increased immigration controls, the extra tariffs, the breakdown in relations, the wars, etc. all push countries towards de-globalization of their supply chains and towards more local manufacturing. What was the root cause of the initial outsourcing? Many reasons, but I would argue that in the end it all comes down to human labor. And it's even more relevant today.

The only way that manufacturing is coming back beyond a total breakdown of imports is via automation. China hasn't been about cheap labor for a long time now. They are automating even faster than the West in part because they learned from the mistakes of the West. Automation is a survival thing. For everyone. If the West does not seriously start to automate manufacturing and fast, in the long run it will become irrelevant. IMO this is where the source of the demand that will feed itself for many decades is coming from. This is also the main bet I'm making with Elench.

### Cost of intelligence

The strongest version of the AI maximalist argument is that eventually some big model with the right scaffolding writes software better than a very talented team of engineers. Maybe. I don't fully buy it yet. But I don't have to.

Even if you discount the maximalist take, we'll have a lot more GPUs running a lot more capable models, and the unit cost of capability will keep dropping. Aggregate spend goes up because we use more (Jevons paradox), but per-unit cost falls enough that training world models, VLAs, and whatever comes next becomes economically viable. That's the input curve for everything Elench is building. Models that can run factories autonomously, recover from failures, inspect parts, and improve from their own data are exactly what gets cheaper as this curve moves. Cheaper intelligence eventually means factories that can build other factories.

### Useful robotics is largely a software problem

From a reliability and cost perspective, five finger hands are not a solved problem. Neither are humanoids or quadrupeds. However, industrial cobot arms with two finger grippers are reliable and cost effective. Their failure modes are well understood and somewhat predictable. Their range of motion is wide enough for a variety of complex tasks. Yet today, most cobot arms run pre-programmed motions with no visual feedback.

This is changing with VLAs and other architectures where you collect a large and diverse dataset and train a model that is able to generalize across a large set of tasks without explicit motion planning. To me what's most striking about VLAs is the ability to recover from an error: dropped a screw on the table? No problem, pick it up and try again.

VLAs are far from perfect though. Long horizon tasks are difficult and fine motor control is still hard. The field is still early. To use a bad analogy: we're roughly where language models were around 2019. Transformers had been invented, BERT was out and crushing benchmarks, but we hadn't hit ChatGPT yet. Same vibes here. I would not bet on physical AI looking the same in 5 years. The data world for physical AI is nothing like the one LLMs grew up in, and the models themselves still need to mature. Closing that gap on both fronts is probably the highest risk factor in my bet with Elench.

### So what is Elench gonna do?

The first question to answer: can we automate the majority of tasks in a 3D printing farm, end to end, running cobot arms with VLA policies fully lights-out for weeks at a time? Not just the obvious parts like swapping build plates and feeding more filament, but the harder stuff: post-processing like removing support material, and minor assembly like pressing screw inserts into finished parts.

If the answer is yes, we can sell large quantities of products into B2B, D2C and defense markets. The same automation stack also extends to other domains beyond plastics.

By the way, print farms exist today and they are run by humans. Filament is maybe 20% to 30% of their opex, the rest is labor and overhead. Take the labor out, reduce logistics cost, and the unit economics flip. Add VLA models that get better with every print and the cost curve compounds.

Then extend the platform. FDM plastic first because it's the easiest place to start, but the same automation principles (manipulation, scheduling, failure recovery, quality inspection) extend to resin, then metal, then PCB fab and assembly. Every step is harder, but every step builds on the previous one's data.

And here's the part where I get a little [Culture](https://en.wikipedia.org/wiki/The_Culture)-pilled.

### Factories that build factories

Once you can automate plastic, metal, and PCBs, you have a strange and powerful property. Those are exactly the categories of components a robot arm or a 3D printer is made of. Plastic housings, machined gears, control boards, motor drivers, sensors. So at some point the factories start producing the hardware for the next generation of factories.

Not a new idea either. [Drexler](https://en.wikipedia.org/wiki/K._Eric_Drexler) was writing about it in the 80s, [von Neumann](https://en.wikipedia.org/wiki/Self-replicating_machine) earlier than that. What's new is that the economics are getting close to making it real.

The honest endgame is more boring and more interesting than singularity stuff. You deploy a factory in NY. Then in Texas. Then in Europe. Then somewhere that needs spare parts and doesn't have a manufacturing base. The factory does the work. Eventually one of the factories builds parts for the next factory. Eventually, hopefully, one of them goes [somewhere humans can't easily go](https://en.wikipedia.org/wiki/Lunar_outpost_(NASA)).

I am not promising you a lunar factory in 2030. I am saying every design decision we're making (containerized, lights-out, autonomous failure recovery, resilient to bad comms) is also a decision that doesn't rule one out. Ten years ago I [argued](https://plugaru.org/2016/04/24/future-of-humanity-and-strong-AI/) that we need autonomous machines to actually colonize space. If that was right, the path there starts with autonomous machines doing useful things on Earth first. You don't get to the asteroid belt and build an orbital without first getting really good at making things in a box.

### About the name

Elench is from the Greek [elenchus](https://en.wikipedia.org/wiki/Socratic_method), the Socratic method of testing a claim by following it to its consequences until it either holds or falls apart. It's also the name of a Culture faction, from one of my favorite [Iain M. Banks](https://en.wikipedia.org/wiki/Iain_Banks) novels, that split off from the mainstream to seek out things stranger than themselves and be changed by them, rather than absorb them and stay the same. Both meanings feel deeply personal and an ethos I want to maintain at Elench.

If this resonates with you, I'm hiring a small founding team of engineers (robot learning, mechatronics, mechanical, etc.). Please reach out to me directly at [alex@elench.com](mailto:alex@elench.com).

Big things have small beginnings.