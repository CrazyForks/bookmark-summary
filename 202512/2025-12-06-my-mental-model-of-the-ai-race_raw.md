Title: My mental model of the AI race

URL Source: https://interconnected.org/home/2025/12/05/training

Markdown Content:
I left a loose end the other day when I said that [AI is about intent and context](https://interconnected.org/home/2025/11/28/plumbing).

That was when I said what’s context at inference time is valuable training data if it’s recorded.

But I left it at that, and didn’t really get into why training data is valuable.

I think we often just draw a straight arrow from _“collect training data,”_ like ingest pages from Wikipedia or see what people are saying to the chatbot, to _“now the AI model is better and therefore it wins.”_

But I think it’s worth thinking about what that arrow actually means. Like, what is the mechanism here?

Now all of this is just my mental model for what’s going on.

With that caveat:

To my mind, the era-defining AI company is the one that is the first to close two self-accelerating loops.

Both are to do with training data. The first is the general theory; the second is specific.

* * *

### Training data for platform capitalism

When I say era-defining companies, to me there’s an era-defining idea, or at least era-_describing,_ and that’s Nick Srnicek’s concept of [Platform Capitalism](https://www.amazon.co.uk/Platform-Capitalism-Theory-Redux-Srnicek/dp/1509504877)_(Amazon)._

It is the logic that underpins the success of Uber, Facebook, Amazon, Google search (and in the future, Waymo).

[I’ve gone on about platform capitalism before](https://interconnected.org/home/2020/12/11/thingscon) (2020) but in a nutshell Srnicek describes a process whereby

*   these companies create a marketplace that brings together buyers and sellers
*   they gather data about what buyers want, what sellers have, how they decide on each other (marketing costs) and how decisions are finalised (transaction costs)
*   then use that data to (a) increase the velocity of marketplace activity and (b) grow the marketplace overall
*   thereby gathering data faster, increasing marketplace efficiency and size faster, gathering data faster… and so on, a runaway loop.

Even to the point that in 2012 [Amazon filed a patent on anticipatory shipping in 2012](https://techcrunch.com/2014/01/18/amazon-pre-ships/)_(TechCrunch)_ in which, if you display a strong intent to buy laundry tabs, they’ll put them on a truck and move them towards your door, only aborting delivery if you end up not hitting the Buy Now button.

And this is also kinda how Uber works right?

Uber has a better matching algorithm than you keeping the local minicab company on speed dial on your phone, which only works when you’re in your home location, and surge pricing moves drivers to hotspots in anticipation of matching with passengers.

And it’s how Google search works.

They see what people click on, and use that to improve the algo which drives marketplace activity, and AdSense keyword cost incentivises new entrants which increases marketplace size.

So how do marketplace efficiency and marketplace size translate to, say, ChatGPT?

ChatGPT can see what success looks like for a “buyer” (a ChatGPT user).

They generate an answer; do users respond well to it or not? (However that is measured.)

So that usage data becomes training data to improve the model to close the gap between user intent and transaction.

Right now, ChatGPT itself is the “seller”. To fully close the loop, they’ll need to open up to other sellers and ChatGPT itself transitions to being the market-maker (and taking a cut of transactions).

And you can see that process with the [new OpenAI shopping feature](https://openai.com/index/chatgpt-shopping-research/) right?

This is the template for all kinds of AI app products: anything that people want, any activity, if there’s a transaction at the end, the model will bring buyers and sellers closer together – marketplace efficiency.

Also there is marketplace size.

Product discovery: OpenAI can see what people type into ChatGPT. Which means they know how to target their research way better than the next company which doesn’t have access to latent user needs like that.

So here, training data for the model mainly comes from usage data. It’s a closed loop.

But how does OpenAI (or whoever) get the loop going in the first place?

With some use cases, like (say) writing a poem, the “seed” training data was in the initial web scrape; with shopping the seed training data came as a result of adding web search to chat and watching users click on links.

But there are more interesting products…

How do product managers triage tickets?

How do plumbers do their work?

You can get seed training data for those products in a couple ways but I think there’s an assumption that what happens is that the AI companies need to trick people out of their data by being present in their file system or adding an AI agent to their SaaS software at work, then hiding something in the terms of service that says the data can be used to train future models.

I just don’t feel like that assumption holds, at least not for the biggest companies.

**Alternate access to seed training data method #1: just buy it.**

I’ll take one example which is multiplayer chat. [OpenAI just launched group chat in ChatGPT](https://openai.com/index/group-chats-in-chatgpt/):

> We’ve also taught ChatGPT new social behaviors for group chats. It follows the flow of the conversation and decides when to respond and when to stay quiet based on the context of the group conversation.

Back in May I did [a deep dive into multiplayer AI chat](https://interconnected.org/home/2025/05/23/turntaking). It’s _really_ complicated. I outlined all the different parts of conversational turn taking theory that you need to account for to have a satisfying multiplayer conversation.

What I didn’t say at the end of that post was that, if I was building it, the whole complicated breakdown that I provided is _not_ what I would do.

Instead I would find a big corpus of group chats for seed data and just train the model against that.

And it wouldn’t be perfect but it would be good enough to launch a product, and then you have actual live usage data coming in and you can iteratively train from there.

Where did that seed data come from for OpenAI? I don’t know. There was that reddit deal last year, maybe it was part of the bundle.

So they can buy data.

Or they can make it.

**Alternate access to seed training data #2: cosplay it.**

Every so often you hear gossip about how seed training data can be manufactured… I remember seeing a tweet about this a few months ago and [now there’s a report](https://x.com/richeholmes/status/1970886134516285587):

> AI agents are being trained on clones of SaaS products.
> 
> 
> According to a new @theinformation report, Anthropic and OpenAI are building internal clones of popular SaaS apps so that they can train AI agents how to use them.
> 
> 
> Internal researchers are giving the agents cloned, fake versions of products like Zendesk and Salesforce to teach the agents how to perform the tasks that white collar workers currently do.

The tweet I ran across was from a developer saying that cloning business apps for the purpose of being used in training was a sure-fire path to a quick acquisition, but that it felt maybe _not ok._

My point is that AI companies don’t need sneak onto computers to watch product managers triaging tickets in Linear. Instead, given the future value is evident, it’s worth it to simply build a simulation of Linear, stuff it with synthetic data, then pay fake product managers to cosplay managing product inside fake Linear, and train off that.

Incidentally, the reason I keep saying _seed_ training data is that the requirement for it is one-off. Once the product loop has started, the product creates it own. Which is why I don’t believe that revenue from licensing social network data or scientific paper is real. There will be a different pay-per-access model in the future.

I’m interested in whether this model extends to physical AI.

Will they need lanyards around the necks of plumbers in order to observe plumbing and to train the humanoid robots of the future?

Or will it be more straightforward to scrape YouTube plumbing tutorials to get started, and then build a simulation of a house (physical or virtual, in Unreal Engine) and let the AI teach itself?

What I mean is that AI companies need access to seed training data, but where it comes from is product-dependent and there are many ways to skin a cat.

* * *

That’s loop #1 – a LLM-mediated marketplace loop that (a) closes on transactions and (b) throws off usage data that improves market efficiency and reveals other products.

Per-product seed training data is a one-off investment for the AI company and can be found in many ways.

This loop produces cash.

* * *

### Coding is the special loop that changes everything

Loop #2 starts with a specific product from loop #1.

A coding product isn’t just a model which is good at understanding and writing code. It has to be wrapped in an agent for planning, and ultimately needs access to collaboration tools, AI PMs, AI user researchers, and all the rest.

I think it’s pretty clear now that coding with an agent is vastly quicker than a human coding on their own. And not just quicker but, from my own experience, I can achieve goals that were previously beyond my grasp.

The loop closes when coding agents accelerate the engineers who are building the coding agents and also, as a side effect, working on the underlying general purpose large language model.

There’s an interesting kind of paperclip maximisation problem here which is, if you’re choosing where to put your resources, do you build paperclip machines or do you build the machines to build the paperclip machines?

Well it seems like all the big AI companies have made the same call right now which is to pile their efforts into accelerating coding, because doing that accelerates everything else.

* * *

So those are the two big loops.

Whoever gets those first will win, that’s how I think about it.

I want to add two notes on this.

* * *

**On training data feeding the marketplace loop:**

Running the platform capitalism/marketplace loop is not the only way for a company to participate in the AI product economy.

Another way is to enable it.

Stripe is doing this. They’re working hard to be the default transaction rails for AI agents.

Apple has done this for the last decade or so of the previous platform capitalism loop. iPhone is the place to reach people for all of Facebook, Google, Amazon, Uber and more.

When I said [before](https://interconnected.org/home/2025/11/28/plumbing) that AI companies are trying to get closer to the point of intent, part of what I mean I that they are trying to figure out a way that a single hardware company like Apple can’t insert itself into the loop and take its 30%.

Maybe, in the future, device interactions will be super commoditised. iPhone’s power is that is bundles together an interaction surface, connectivity, compute, identity and payment, and we have one each. It’s interesting to imagine what might break that scarcity.

* * *

**On coding tools that improve coding tools:**

How much do you believe in this accelerating, self-improving loop?

The big AI research labs all believe – or at least, if they don’t believe, they believe that the risk of being wrong is worse.

But, if true, “tools that make better tools that allow grabbing bigger marketplaces” is an Industrial Revolution-like driver: technology went from the steam engine to the transistor in less than 200 years. Who knows what will happen this time around.

Because there’s a _third_ loop to be found, and that’s when the models get so good that they can be used for novel R&D, and the AI labs (who have the cash and access to the cheapest compute) start commercialising [wheels with weird new physics](https://interconnected.org/home/2022/03/02/wheels) or whatever.

Or maybe it’ll stall out. Hard to know where the top of the S-curve is.