Title: What happens when the coding becomes the least interesting part of the work

URL Source: https://obie.medium.com/what-happens-when-the-coding-becomes-the-least-interesting-part-of-the-work-ab10c213c660

Published Time: 2025-12-12T09:07:43Z

Markdown Content:
[![Image 1: Obie Fernandez](https://miro.medium.com/v2/resize:fill:64:64/1*HBnYc_3J-_5uWc2Sbnx0Jg.jpeg)](https://obie.medium.com/?source=post_page---byline--ab10c213c660---------------------------------------)

8 min read

2 days ago

--

S omething that’s been true for me for a long time, long before coding agents showed up, is that the initial effort involved in any programming work happens before the first line of code is written. I say this as someone who routinely “thinks with his fingers,” as they say, but it’s not something I’ve had to think about until lately now that I’m [pair programming all the time](https://medium.com/@obie/ruby-was-ready-from-the-start-4b089b17babb) again.

There’s a problem to be solved. Maybe it’s a new feature, or maybe it’s a bug fix. I familiarize myself with the problem. My brain immediately starts pattern recognition, searching the solution space against decades of learning. Once the solution is obvious, which is quite often, the rest (the actual coding part) is mostly follow-through.

That doesn’t mean the follow-through is unimportant. It just means it stopped being the thing that teaches me anything, and that makes it _boring_. Which is why using coding agents is such a game-changer for me, but it might not be for you. But before I go there, let me back up a second and explain what I’m talking about a bit better.

The Silent Checklist That Runs Before I Type
--------------------------------------------

When I pick up a task, a bunch of questions fire almost immediately. They’re not formal. I don’t write them down. Half the time I wouldn’t even know how to explain them unless someone asked.

*   What kind of change is this, really?
*   Is this introducing a new idea, or just expressing an existing one in a new place?
*   Does this feel like it belongs close to the surface (API, UX, etc) or deep inside the system?
*   If this thing I’m planning to introduce or change (whatever it is) spreads, will that be a good thing or a liability?
*   What happens if we’re wrong and need to undo it? What’s the _blast radius?_

Note that none of those questions mention code explicitly. They’re about shape, direction, and consequences of the engineering work I’m undertaking. Code is only part of the equation.

Questions of appropriate abstraction level show up here sometimes, but only as one signal among many. Sometimes the right move is to generalize. Sometimes the right move is to actively resist it. Sometimes the right move is to do something slightly ugly because it keeps the decision reversible.

That judgment is the job of a senior engineer. As far as I can tell, nobody is replacing that job with a coding agent anytime soon. Or if they are, they’re not talking about it publicly. I think it’s the former, and one of the main reasons is that there is probably not that much spelled-out practical knowledge about how to do the job of a senior engineer in frontier models’ base knowledge, the stuff that they get from their primary training by ingesting the whole internet.

But wait a minute, aren’t there plenty of books written on the topic? They’re probably included in the training right? Martin Fowler, Uncle Bob, etc? As someone that’s read many of those books, I can tell you that they only scrape the surface of what a senior engineer actually brings to bear on an active project everyday. The same probably applies to the mass of blog posts that have surely been written on the subject over the years, including my own.

Press enter or click to view image in full size

Why the most important stuff stays invisible
--------------------------------------------

For the sake of discussion, let’s call this stuff _senior thinking._ Most of this thinking happens too fast and too quietly to get shared methodically.

If I’m working alone, it all stays internal. If I’m pairing with someone else that’s senior, then the thinking surfaces naturally in conversation but is taken for granted. If I’m pairing with someone less experienced, I might slow down enough to explain parts of my senior thinking, but definitely not all. It would bog us down too much.

Teams rarely standardize or document senior thinking because 1) it feels hard to pin down and 2) it’s practically boundless. If anything, it’s easier to talk about patterns and frameworks than about instinct and timing, but the implementation of those patterns and understanding of the chosen frameworks are considered to be part of the job. (As I write this it occurs to me that it is the single biggest reason that bad code exists.)

Which leads me to the point of this blog post. Coding agents such as Claude Code with Opus 4.5, are turning out to be even more interesting than I expected, for reasons that hadn’t really fully clicked until last night. I’ve been in Abu Dhabi for the Solana Breakpoint conference for the last couple days (yes, I’m taking another foray in cryptolandia!) and I’m severely jetlagged. Luckily my brain sometimes treats insomnia as a freewheeling playground for blog ideas.

Press enter or click to view image in full size

Agents force you to say the unspoken parts out loud
---------------------------------------------------

When you pair program with Claude Code or any other coding agent for that matter, you have to explain what you want. Not just what to build, but _how you’re thinking about the problem_. If you only describe the what and not the how, expect to be underwhelmed.

You can’t just let the AI guess the correct abstraction level, you have to specify it. You can’t just sit there thinking that a change is risky if you want the AI to take that risk into account. You have to describe it out loud. Similarly, you can’t just let the AI figure out that something you’re asking for is meant to be a temporary kluge. You have to explicitly frame it that way.

Here’s where this new way of working is making me _better_ not worse, and why I’m not worried about what the use of AI is doing to our collective ability to do senior thinking, assuming that you use coding agents the way that I do, that is, by pair programming with them.

That act of explanation does something important. It slows you down enough to notice when your instincts are off. It also makes your reasoning visible in a way that solo work rarely does. That is exactly the same effect that working with a good pair programming partner has always had.

I don’t expect this situation to change dramatically in the near future, not without some real AGI in the mix, and here’s why.

The senior toolkit is bigger than people think
----------------------------------------------

Levels of abstraction and generalization problems get talked about a lot because they’re easy to name. But they’re far from the whole story.

Other tools show up just as often in real work:

*   A sense for blast radius. Knowing which changes are safe to make loudly and which should be quiet and contained.
*   A feel for sequencing. Knowing when a technically correct change is still wrong because the system or the team isn’t ready for it yet.
*   An instinct for reversibility. Preferring moves that keep options open, even if they look less elegant in the moment.
*   An awareness of social cost. Recognizing when a clever solution will confuse more people than it helps.
*   An allergy to false confidence. Spotting places where tests are green but the model is wrong.

None of that fits neatly into a framework. Unless someone goes back in time and secretly records all the pair programming sessions that went on at Thoughtworks and Pivotal Labs and Hashrocket, there’s literally no significant corpus/written record of this kind of spoken out loud thinking in existence. (Fuck, it would be so valuable though!)

At the moment senior thinking is only learned over the course of years and decades by watching systems bend and sometimes break depending on how you write and maintain.

Coding agents are definitely aware of what the elements of senior thinking are and what they mean, but have no intrinsic determination to apply them to the work that they do. Could they gain that determination via reinforcement learning or better reward functions? Anything’s possible, I guess. But I don’t exactly expect any of us who posess this knowledge to line up to share it willingly so that we can be replaced. I hope not, anyway.

Press enter or click to view image in full size

Why this subject flies right over some of my colleagues heads
-------------------------------------------------------------

I’ve met so many programmers who haven’t even started using Claude Code yet. Or have tried and just didn’t see the point. Or even worse, didn’t like it! And as I consider this whole issue of senior thinking, I think I’m starting to understand some of them.

If the part of programming you enjoy most is the physical act of writing code, then agents will feel beside the point. You’re already where you want to be, even just with some Copilot or Cursor-style intelligent code auto completion, which makes you faster while still leaving you fully in the driver’s seat about the code that gets written.

But if the part you care about is the decision-making around the code, agents feel like they clear space. They take care of the mechanical expression and leave you with judgment, tradeoffs, and intent. Because truly, for someone at my experience level, that is my core value offering anyway. When I spend time actually typing code these days with my own fingers, it feels like a waste of my time. If that time is being billed to my client at my expensive hourly rate, I feel even worse about it.

For a lot of other people, especially if they’re less senior, the reaction to coding agents is nowhere near that dramatic. It’s not that they’re disagreeing with me, assuming they’re even paying attention to this kind of thing at all. It’s just not the part of the job that they care about. What I would warn them about though, is that what they care most about is the part of the job that can _already_ be replaced by AI. In fact, what is probably holding that back from happening faster across the industry is either an overall lack of senior talent, or more likely, simply that senior talent not quite yet figuring out what is going on because they are not using coding agents the way that I’m describing.

Assuming that last part is accurate, I wouldn’t expect the situation to last much longer though — because once senior engineers do start figuring it out en masse, and in particular, once their employers figure it out, it’s going to be a bloodbath out there. Even the most productive juniors and mids are going to start feeling like net negatives. The blast radius is not limited to just engineers either. If I can cut a 20 person team that I manage down to 4 or 5 highly accountable and AI empowered senior engineers that report directly to me, then I don’t even need a project manager. I might not even need business analysts, because the seniors can be my domain experts. Depending on the domain, I might not even need designers.

[](https://x.com/emollick/status/1999242260945178813?s=20)

[https://x.com/emollick/status/1999242260945178813](https://x.com/emollick/status/1999242260945178813?s=20)

The bottom line: even if current LLM progress hits a brick wall at Opus 4.5 level (and I doubt that will happen) the next 12 months are still going to be a staggering time of change in this industry as decision makers start truly understanding the new reality we live in.