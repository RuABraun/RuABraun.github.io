---
layout: post
title: The do everything abstraction
date: 2021-04-12 12:06:02 +0200
hidden: true
categories: jekyll update
---


Premature abstraction is something most people are aware of but I think a more common mistake is the "do everything" abstraction.

Here's what happens:
A coder needs code that accomplishes a task. They think "cool let me create a class/function that does what I need".

Sometime later they finish coding it up, pat themselves on the back and move on.

A week/month/year later it turns out that solution contains a step (one of several) which would be useful on its own. Oh and the final step is also something that would be nice to use. At that point it would be good to refactor.

The reason this happens is because it's easy and tempting to think I need code that does "X", without realizing that doing "X" actually consists of doing "A", "B", "C" and that these individual transformations are useful by themselves and therefore should be separated out.

Of course this requires having some experience to judge what tasks could be needed and whether the code is for longterm use and so on.

But in general this is the most common abstraction mistake, going too high-level.

The opposite can also happen, forcing a user (coder) to call many low-level abstractions repeatedly although they always call the same ones. But this mistake is rarer and (usually) less disruptive.

Choosing the right level of abstraction is difficult, but I think it's something a coder should ask themselves when developing something. Who are my users? Do they do one thing 50% of the time and 4 other things the remaining 50? Or do they do one thing 95% of the time? What are the things they could reasonably want to do? Could what they want change?

Judging these things is hard. It's so easy to have a flaw in one's logic that leads one to misjudging the validity of a usecase.

Because of this difficulty it's best to just do whatever is simple first because simple is easy to refactor. The "do-everything" abstraction may be right if it does the job and can be easily refactored to support more use-cases. But it must be simple! If one can't make a "do-everything" abstraction without it being complex then that's a big red warnings sign that what you are doing is bad and you should spend some time thinking about what the core tasks are (what are "A", "B" and "C"?).
