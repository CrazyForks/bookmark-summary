Title: GraphQL: the enterprise honeymoon is over

URL Source: https://johnjames.blog/posts/graphql-the-enterprise-honeymoon-is-over

Published Time: 2025-12-14T16:48:21Z

Markdown Content:
GraphQL: the enterprise honeymoon is over
===============
![Image 1: graphql-the-enterprise-honeymoon-is-over](https://johnjames.blog/_next/image?url=https%3A%2F%2Fcdn.sanity.io%2Fimages%2Fgq0n5o36%2Fproduction%2Fb78d3be6a3c86ba719624ea47ddf70541542b173-1536x1024.png%3Frect%3D0%2C224%2C1536%2C576%26w%3D800%26h%3D300%26q%3D80%26auto%3Dformat&w=1920&q=80)
GraphQL: the enterprise honeymoon is over
=========================================

By John James

Published on December 14, 2025

Read time ~3 min

I’ve used GraphQL, specifically Apollo Client and Server, for a couple of years in a real enterprise-grade application.

Not a toy app. Not a greenfield startup. A proper production setup with multiple teams, BFFs, downstream services, observability requirements, and real users.

And after all that time, I’ve come to a pretty boring conclusion:

GraphQL solves a real problem, but that problem is far more niche than people admit. In most enterprise setups, it’s already solved elsewhere, and when you add up the tradeoffs, GraphQL often ends up being a net negative.

This isn’t a “GraphQL bad” post. It’s a “GraphQL after the honeymoon” post.

### what GraphQL is supposed to solve

The main problem GraphQL tries to solve is overfetching.

The idea is simple and appealing:

*   the client asks for exactly the fields it needs
*   no more, no less
*   no wasted bytes
*   no backend changes for every new UI requirement

On paper, that’s great. In practice, things are messier.

### overfetching is already solved by BFFs

Most enterprise frontend architectures already have a BFF (Backend for Frontend).

That BFF exists specifically to:

*   shape data for the UI
*   aggregate multiple downstream calls
*   hide backend complexity
*   return exactly what the UI needs

If you’re using REST behind a BFF, overfetching is already solvable. The BFF can scope down responses and return only what the UI cares about.

Yes, GraphQL can also do this. But here’s the part people gloss over.

Most downstream services are still REST.

So now your GraphQL layer still has to overfetch from downstream REST APIs, then reshape the response. You didn’t eliminate overfetching. You just moved it down a layer.

That alone significantly diminishes GraphQL’s main selling point.

There is a case where GraphQL wins here. If multiple pages hit the same endpoint but need slightly different fields, GraphQL lets you scope those differences per query.

But let’s be honest about the trade.

You’re usually talking about saving a handful of fields per request, in exchange for:

*   more setup
*   more abstraction
*   more indirection
*   more code to maintain

That’s a very expensive trade for a few extra kilobytes.

### implementation time is much higher than REST

GraphQL takes significantly longer to implement than a REST BFF.

With REST, you typically:

*   call downstream services
*   adapt the response
*   return what the UI needs

With GraphQL, you now have to:

*   define a schema
*   define types
*   define resolvers
*   define data sources
*   write adapter functions anyway
*   keep schema, resolvers, and clients in sync

GraphQL optimizes consumption at the cost of production speed.

In an enterprise environment, production speed matters more than theoretical elegance.

### observability is worse by default

This one doesn’t get talked about enough.

GraphQL has this weird status code convention:

*   400 if the query can’t be parsed
*   200 with an `errors` array if something failed during execution
*   200 if it succeeded or partially succeeded
*   500 if the server is unreachable

From an observability standpoint, this is painful.

With REST:

*   2XX means success
*   4XX means client error
*   5XX means server error

If you filter dashboards by 2XX, you know those requests succeeded.

With GraphQL, a 200 can still mean partial or full failure.

Yes, Apollo lets you customize this behavior. But that’s kind of the point. You’re constantly paying a tax in extra configuration, extra conventions, and extra mental overhead just to get back to something REST gives you out of the box.

This matters when you’re on call, not when you’re reading blog posts.

### caching sounds amazing until you live with it

Apollo’s normalized caching is genuinely impressive.

In theory. In practice, it’s fragile.

If you have two queries where only one field differs, Apollo treats them as separate queries. You then have to manually wire things so:

*   existing fields come from cache
*   only the differing field is fetched

At that point:

*   you still have a roundtrip
*   you’ve added more code
*   debugging cache issues becomes its own problem

Meanwhile, REST happily overfetches a few extra fields, caches the whole response, and moves on. Extra kilobytes are cheap. Complexity isn’t.

### the ID requirement is a leaky abstraction

Apollo expects every object to have an `id` or `_id` field by default, or you need to configure a custom identifier.

That assumption does not hold in many enterprise APIs.

Plenty of APIs:

*   don’t return IDs
*   don’t have natural unique keys
*   aren’t modeled as globally identifiable entities

So now the BFF has to generate IDs locally just to satisfy the GraphQL client.

That means:

*   more logic
*   more fields
*   you’re always fetching one extra field anyway

Which is ironic, considering the original goal was to reduce overfetching.

REST clients don’t impose this kind of constraint.

### file uploads and downloads are awkward

GraphQL is simply not a good fit for binary data.

In practice, you end up:

*   returning a download URL
*   then using REST to fetch the file anyway

Embedding large payloads like PDFs directly in GraphQL responses leads to bloated responses and worse performance.

This alone breaks the “single API” story.

### onboarding is slower

Most frontend and full-stack developers are far more experienced with REST than GraphQL.

Introducing GraphQL means:

*   teaching schemas
*   teaching resolvers
*   teaching query composition
*   teaching caching rules
*   teaching error semantics

That learning curve creates friction, especially when teams need to move fast.

REST is boring, but boring scales extremely well.

### error handling is harder than it needs to be

GraphQL error responses are… weird.

You have:

*   nullable vs non-nullable fields
*   partial data
*   errors arrays
*   extensions with custom status codes
*   the need to trace which resolver failed and why

All of this adds indirection.

Compare that to a simple REST setup where:

*   input validation fails, return a 400
*   backend fails, return a 500
*   zod error, done

Simple errors are easier to reason about than elegant ones.

### the net result

GraphQL absolutely has valid use cases.

But in most enterprise environments:

*   you already have BFFs
*   downstream services are REST
*   overfetching is not your biggest problem
*   observability, reliability, and speed matter more

When you add everything up, GraphQL often ends up solving a narrow problem while introducing a broader set of new ones.

That’s why, after using it in production for years, I’d say this:

GraphQL isn’t bad. It’s just niche. And you probably don’t need it.

Especially if your architecture already solved the problem it was designed for.

* * *

### see also

*   [cleaning house in nx monorepo, how i removed 120 unused deps safely](https://johnjames.blog/posts/cleaning-house-in-nx-monorepo-how-i-removed-120-unused-deps-safely)
*   [automating dependabot reviews, how AI cut 95% of dependency research time](https://johnjames.blog/posts/automating-dependabot-reviews-how-ai-cut-95-percent-of-dependency-research-time)