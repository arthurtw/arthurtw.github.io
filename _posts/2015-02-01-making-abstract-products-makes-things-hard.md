---
title: Making Abstract Products Makes Things Hard
layout: post
---

Software industry is unique and charming by making abstract products, but it’s also where the hardship comes from. In this article, I am going to share my thoughts on this unique nature and its consequence.

## Software is abstract product

First of all, we shall distinguish products from services. It’s an interesting exercise. (E.g. what does Google, Apple or Amazon provide?) Think about it for a moment.

#### Product vs. service

In general, products are transferable things and services are human interactions. They often accompany each other. For example, you request a painter to paint a portrait for you. Are you paying for a product or a service? Actually, it’s both: the painting is a product, and fulfilling your request is a service. Note that you can transfer a product you own (the painting), but not a service you received.

Another example is public transportation. When you take a bus, both a service (driving) and a product (bus) are involved, though you cannot transfer the product because you only paid for its temporary shared use.

A business can provide products, services, or both.

#### Software as a service

SaaS is a popular term these days. Although it’s described as a service, I think the “software” here is a product. It’s just another way of product monetization like joint tenancy. (Do not confuse it with customer service, which is often part of the product provision.) Companies may run business and provide services on top of some software, but the software itself is a product.

Why do I keep talking on this? Because the skills of making products and providing services are fundamentally different. They overlap at some area, e.g. knowing your customers, but *making great products requires great understanding of how to make them*. Sounds prattle? No it’s not. In fact, I think many problems in the software industry, business or management come from the lack of such understanding, and it’s hard because we are making abstract products.

#### The product abstractness is unique

Except software, what else products are abstract? The abstract works done by scientists or researchers? Those are explorations and findings for someone else to look at. Creations like music, arts or movies? They are experienceable artistic work like sculptures, just in different forms. Semiconductor devices are probably the closest, but they are concrete products with physical existence.

“But software has UI”, you may say. Indeed, and that’s exactly why many non-engineering stakeholders (customers, product managers, executives etc.) focus on UI and features heavily, because that’s the easiest part to grasp. Unfortunately, in software making, it’s often the tip of the iceberg. Beyond that, things quickly become obscure where only a few people can see through.

#### The ultimate form of software

Here is another interesting question: what is software ultimately? I think it’s instructions to machine, typically via CPU to hardware. Computer is a state machine, and software instructs its state transitions.

Because of the unmanageable complexities, we have to build software on top of layers of abstractions.

#### Layers of software abstractions

Software today is made with many layers of abstractions. The OS along with device drivers is usually the first layer, sitting on top of hardware. The programming languages we use are higher-level, sitting on top of compilers, interpreters and other runtime environments. Networking or collaborating with other computing devices is another source of massive complexities. Our code uses many libraries and frameworks to cope with the ever-increasing complexities nowadays. (Think about web frontend as an example. In early days, static web pages or CGI scripts sufficed; then we got Ajax and rich web applications for better user experience, then responsive web design for multiple devices, then isomorphic JavaScript for server-side rendering...)

The software products we make usually consist of code in different programming or markup languages. The code *is* our product.

## Making abstract products is hard

We have been making tangible products for thousands of years and accumulated numerous experiences on that. When making abstract products, however, many rules don’t apply and things get harder.

#### Complexities may go wild

Imagine a mathematician is writing multi-page arcane formulas to solve a problem in a particular math domain. Most people would be averse to reading them unless you happen to be 1) good at math and 2) familiar with that domain. If it’s hundreds of pages thick instead of several pages, written by various mathematicians to solve numerous problems instead of one, that would be very complicated. And our software is just like, or more complicated than, that.

The complexities of a physical product cannot go unlimitedly wild due to physical constraints and cost. 

#### Code reading is mind reading

The code is the reflection of the coder’s mental working. To understand the code is to understand (or guess) the coder’s mind. That can be very difficult. The worse the code, the harder. Unfortunately, bad code is far more common than good code.

Code has an interesting character duo: it’s written for *both* human and machine. Yet unlike writing articles or books while the writer often keeps readers in mind, a coder often forgets to.

#### No visual mapping

Software has no proper visual mapping or presentation for human’s observation. You cannot “see” software. That makes communication and understanding very hard. You can draw some charts, but they are far from the actual code.

Instead, you have to form your comprehension of the code in your head. It causes many problems: such comprehension is hard to form, easy to forget, and inconsistent between different people or time. It also makes developers’ context switch expensive. (I’ll talk more about context switch later.)

#### Build quality relies on self-discipline

The observation difficulties also make software build quality hard to control. Code that works is not necessarily well built. For example, developers can copy and paste lousily or write tangled code, and still deliver all features on time. Steve Jobs might have insisted on the beauty of Mac circuit board, but hardly on the beauty of the code. (“Boss wants beautiful code? I gotta pretty-print it.”)

Unlike concrete products, adding more code incurs no material cost. Even code review or metrics like test coverage rely on the development team’s self-discipline. (Test coverage can be off, and the quality of test code can be another source of problems itself.)

## Management challenges

#### You quickly lose track after two levels

It’s already hard for a tech lead to keep track of all team members’ code. Moving one more level up, even you were once a great coder, it becomes almost impossible. You then rely on metrics and updates from your team leads instead of your first-hand observation.

Seeing is believing. It cannot be truer for software code. A good coder can spot many problems with merely a glimpse of the code. It’s a pity to lose track of your product (the code) after two levels, but that’s the way it is.

#### Be careful with metrics and processes

Management often relies on metrics or dashboard for software projects. They are useful to some extent, but can be misleading, too. Metrics only reveal partial truth, and things not tracked by metrics might get de-prioritized automatically. So be very careful.

Good processes aid the team, yet counterproductive processes are everywhere. Even a well-recognized process or methodology can be a productivity killer if it’s wrongly applied. A common problem is those who decide often do not understand all the intricacies and consequences.

#### Context switch is costly

Context switch is costly due to software abstractness and the difficulties to form your comprehension of the code in your head. Even to the same developer working on the same code, it may take him or her a while to “warm up” and get to full speed next morning. (My little trick is to leave an easy part unfinished with some notes to remind myself, so next day I can enter the “zone” faster.)

For the sake of developers efficiency, we should reduce their interruptions, especially those that force context switch. Working on multiple issues or projects simultaneously (“multi-tasking”) is also inefficient. It won’t get things done faster, just distractions.

#### Engineers are not swappable resources

There seems an unhealthy trend in the industry to regard engineers as swappable resources and disregard the context switch cost. In a law firm, it would be unwise to arbitrarily swap lawyers, because the switch cost between different cases is high. (Though lawyers and coders are different, the requirements of loading complex context before start-up are similar.)

Not only are developers moved between different teams frequently (due to e.g. resources arrangement or org change), but they’re also moved between roles of different skill sets (frontend, backend, database, dev-ops etc.) sometimes. I’m not saying that developers should stick to the same team in the same role for years, which is unhealthy either, but such move should be considered individually, and the context switch cost needs to be taken into account.

I’ve seen projects transitioned to new teams and the important nuances got lost completely. The solid hands-on experiences cannot be transferred in a few sessions. An ideal situation is the team stays while team members come and go steadily (i.e. not a clean slate), though it’s not always possible.

#### Great engineers work for great engineers

Great engineers work for great engineers, because they want to work for someone whom they respect and who appreciates their work. It’s discouraging when your management can only recognize the “achievements” listable in their weekly (monthly) reports, or when they do stupid things too often.

The situations in a startup and an established company may vary, but it’s rather hard to hire and retain great engineers without great engineering leaders. It’s also a cascading effect: great engineering leaders work for great engineering leaders. An interesting result is the engineering caliber cap of a company, whether a startup or a well-established one, usually equals that of its founder or founding team.

#### Building abstract products is fascinating

In spite of all the hardship, building abstract products is fascinating. All you do is to type in code. It’s amazing that such an abstract, somewhat imaginary thing can make a difference to people’s life.

Hope my sharing in this article, albeit discrete, has added spice and provoked some thoughts for you.

