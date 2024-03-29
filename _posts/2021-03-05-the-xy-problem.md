---
title: The XY Problem, or how to stay out of tech support purgatory
date: 2021-03-05
tags:
  - General
  - Tech
---

Have you ever been on the giving or receiving end of a marathon troubleshooting session? One that kept everyone going in circles for days, until someone finally stumbled on a solution *by accident*? If you work in IT, the answer is either "yes" or "daily". Some situations are messy no matter what- but others are easy to solve, if you are aware of the **XY Problem** and how to avoid it.

The XY Problem, as originally explained [here](http://mywiki.wooledge.org/XyProblem) and paraphrased [here](https://xyproblem.info/):

> * User wants to do X.
> * User doesn't know how to do X, but thinks they can fumble their way to a solution if they can just manage to do Y.
> * User doesn't know how to do Y either.
> * User asks for help with Y.
> * Others try to help user with Y, but are confused because Y seems like a strange problem to want to solve.
> * After much interaction and wasted time, it finally becomes clear that the user really wants help with X, and that Y wasn't even a suitable solution for X.

> The problem occurs when people get stuck on what they believe is the solution and are unable step back and explain the issue in full.

When people call you for troubleshooting help, they are often convinced that their current solution will work, as long as a few lingering issues are addressed. Unfortunately, their solution may be a dead end, or it may be built on incorrect assumptions- but you won't know that until you understand their original problem. The customer wants to solve for Y, but you have to solve for X.

When browsing Stackoverflow or vendor support forums, you will occasionally see someone ask the magic question:

> What is the problem you are trying to solve?

This is the most direct way to ask: "What is the X behind your Y?" The above question has some sharp edges that should be rounded off though, because asking it directly tends to make people defensive. *What do you mean, 'What is the problem'? I just told you my problem!* Let's soften it with some verbal judo:

> Can you help me understand what the original need is behind this? What goals are you trying to accomplish with what you have now?

## A real-world example

For years, a customer had been using a screen-scraping program to harvest data from our website. We didn't have an API for this type of data, so the customer relied on a messy homebrew solution that was just functional enough to get what they needed. One day, a code change on our website caused their scripts to stop working- and the customer was stumped. In the name of customer service, promises were made that we would support their homebrew screen-scraping method. *Oops*.

Due to that commitment, our early focus was around fixing the customer's scripts- but after a month or two of stab-in-the-dark troubleshooting, the customer was getting frustrated. *Why can't you solve our Y?*

Eventually we were pulled into an all-hands discussion on the issue. That is when the magic question popped up: *What does the customer actually need from us?* What the customer didn't know, was that we have a team who regularly *pushes* the same data to customers who express a need for it. When the customer came on the call, we started off with the magic question. The conversation went like this:

> * **Customer**: So I figured out how to grab a new session token in my script, but I still get stuck at the next step...
> * **Us**: What data are you trying to scrape? What do you need?
> * **Customer**: Um... I need items foo and bar.
> * **Us**: We have a team that can push that data to you on a daily basis. Would that work?
> * **Customer**: Oh my god, yes! That would be awesome!
> * **Us**: OK, we will set up an account and give you details.

After two months of well-intentioned but misguided troubleshooting, the issue evaporated as soon as someone asked the magic question. The solutions aren't always that tidy, but it's pretty satisfying when they are!