Title: I ported JustHTML from Python to JavaScript with Codex CLI and GPT-5.2 in 4.5 hours

URL Source: https://simonwillison.net/2025/Dec/15/porting-justhtml/

Markdown Content:
15th December 2025

I [wrote about JustHTML yesterday](https://simonwillison.net/2025/Dec/14/justhtml/)—Emil Stenström’s project to build a new standards compliant HTML5 parser in pure Python code using coding agents running against the comprehensive html5lib-tests testing library. Last night, purely out of curiosity, I decided to try **porting JustHTML from Python to JavaScript** with the least amount of effort possible, using Codex CLI and GPT-5.2. It worked beyond my expectations.

#### TL;DR[#](https://simonwillison.net/2025/Dec/15/porting-justhtml/#tl-dr)

I built [simonw/justjshtml](https://github.com/simonw/justjshtml), a dependency-free HTML5 parsing library in JavaScript which passes 9,200 tests from the html5lib-tests suite and imitates the API design of Emil’s JustHTML library.

It took two initial prompts and a few tiny follow-ups. [GPT-5.2](https://simonwillison.net/2025/Dec/11/gpt-52/) running in [Codex CLI](https://github.com/openai/codex) ran uninterrupted for several hours, burned through 1,464,295 input tokens, 97,122,176 cached input tokens and 625,563 output tokens and ended up producing 9,000 lines of fully tested JavaScript across 43 commits.

Time elapsed from project idea to finished library: about 4 hours, during which I also bought and decorated a Christmas tree with family and watched the latest Knives Out movie.

#### Some background[#](https://simonwillison.net/2025/Dec/15/porting-justhtml/#some-background)

One of the most important contributions of the HTML5 specification ten years ago was the way it precisely specified how _invalid_ HTML should be parsed. The world is full of invalid documents and having a specification that covers those means browsers can treat them in the same way—there’s no more “undefined behavior” to worry about when building parsing software.

Unsurprisingly, those invalid parsing rules are pretty complex! The free online book [Idiosyncrasies of the HTML parser](https://htmlparser.info/) by Simon Pieters is an excellent deep dive into this topic, in particular [Chapter 3. The HTML parser](https://htmlparser.info/parser/).

The Python [html5lib](https://github.com/html5lib/html5lib-python) project started the [html5lib-tests](https://github.com/html5lib/html5lib-tests) repository with a set of implementation-independent tests. These have since become the gold standard for interoperability testing of HTML5 parsers, and are used by projects such as [Servo](https://github.com/servo/servo) which used them to help build [html5ever](https://github.com/servo/html5ever), a “high-performance browser-grade HTML5 parser” written in Rust.

Emil Stenström’s [JustHTML](https://github.com/EmilStenstrom/justhtml) project is a pure-Python implementation of an HTML5 parser that passes the full html5lib-tests suite. Emil [spent a couple of months](https://friendlybit.com/python/writing-justhtml-with-coding-agents/) working on this as a side project, deliberately picking a problem with a comprehensive existing test suite to see how far he could get with coding agents.

At one point he had the agents rewrite it based on a close inspection of the Rust html5ever library. I don’t know how much of this was direct translation versus inspiration (here’s Emil’s [commentary on that](https://news.ycombinator.com/item?id=46264195#46267059))—his project has 1,215 commits total so it appears to have included a huge amount of iteration, not just a straight port.

My project **is** a straight port. I instructed Codex CLI to build a JavaScript version of Emil’s Python code.

#### The process in detail[#](https://simonwillison.net/2025/Dec/15/porting-justhtml/#the-process-in-detail)

I started with a bit of mise en place. I checked out two repos and created an empty third directory for the new project:

cd ~/dev
git clone https://github.com/EmilStenstrom/justhtml
git clone https://github.com/html5lib/html5lib-tests
mkdir justjshtml
cd justjshtml

Then I started Codex CLI for GPT-5.2 like this:

codex --yolo -m gpt-5.2

That `--yolo` flag is a shortcut for `--dangerously-bypass-approvals-and-sandbox`, which is every bit as dangerous as it sounds.

My first prompt told Codex to inspect the existing code and use it to build a specification for the new JavaScript library:

> `We are going to create a JavaScript port of ~/dev/justhtml - an HTML parsing library that passes the full ~/dev/html5lib-tests test suite. It is going to have a similar API to the Python library but in JavaScript. It will have no dependencies other than raw JavaScript, hence it will work great in the browser and node.js and other environments. Start by reading ~/dev/justhtml and designing the user-facing API for the new library - create a spec.md containing your plan.`

I reviewed the spec, which included a set of proposed milestones, and told it to add another:

> `Add an early step to the roadmap that involves an initial version that parses a simple example document that is valid and returns the right results. Then add and commit the spec.md file.`

Here’s [the resulting spec.md file](https://github.com/simonw/justjshtml/blob/19b8eb1f2ca80f428a3c40862d5ec05d36e5166b/spec.md). My request for that initial version became “Milestone 0.5” which looked like this:

> **Milestone 0.5 — End-to-end smoke parse (single valid document)**
> 
> 
> *   Implement the smallest end-to-end slice so the public API is real early: 
>     *   `new JustHTML("<html><head></head><body><p>Hello</p></body></html>")` returns a tree with the expected tag structure and text nodes.
>     *   `doc.toText()` returns `"Hello"` and `doc.errors` is empty for this valid input.
> 
> *   Add `scripts/smoke.js` (no deps) that runs the example and asserts the expected structure/output.
> *   Gate: `node scripts/smoke.js` passes.

Then I told it:

> `Implement Milestone 0.5`

And off it went. The resulting code appeared to work so I said:

> `Configure GitHub Actions test.yml to run that on every commit, then commit and push`

I ran `git init` and created a private GitHub repository for this project at this point, and set up the local directory to push to that remote.

Here’s [that initial push](https://github.com/simonw/justjshtml/commit/48497cbd05584c88af757a0736a2ee447468b6b3). Then I told it:

> `OK do the rest, commit and push often`

And that was almost it!

I set my laptop to not fall asleep and left it to its devices while we went off to buy a Christmas tree. The “commit and push often” meant I could monitor its progress on my phone by refreshing [the commit log on GitHub](https://github.com/simonw/justjshtml/commits/main).

I was running this against my $20/month ChatGPT Plus account, which has a five hour token allowance window for Codex CLI. That ran out at 6:35pm and Codex paused, so I waited until the reset point at 7:14pm and typed:

> `Keep going`

At 9:30pm it declared itself done with the following summary message:

```
• What’s Done

  - html5lib-tests now run in CI and pass: tokenizer 6810/6810, tree 1770/1782
    (12 skipped: #script-on), serializer 230/230, encoding 82/83 (1 skipped:
    scripted) (.github/workflows/test.yml:1).
  - Public API expanded: stream() (src/stream.js:1), CSS selectors query()/
    matches() (src/selector.js:1), and toMarkdown() (src/markdown.js:1), all
    exported via src/index.js:1.
  - Added no-deps unit scripts for selectors/stream/markdown (scripts/run-
    selector-tests.js:1, scripts/run-stream-tests.js:1, scripts/run-markdown-
    tests.js:1) and wired them into CI.
  - Fixed TreeBuilder so <br> no longer closes <p> (src/treebuilder.js:647).
```

As a finishing touch, I had it add a playground interface so I could try out the new library in my browser. I prompted:

> `Add a playground.html in the top level folder which loads the necessary ES modules from ./src/... and implements the exact same functionality as seen on https://tools.simonwillison.net/justhtml but using the JavaScript library instead of Pyodide`

It fetched my [existing JustHTML playground page](https://tools.simonwillison.net/justhtml) ([described here](https://simonwillison.net/2025/Dec/14/justhtml/#first-impressions-of-justhtml)) using `curl` and built a new `playground.html` file that loaded the new JavaScript code instead. This worked _perfectly_.

I enabled GitHub Pages for my still-private repo which meant I could access the new playground at this URL:

[https://simonw.github.io/justjshtml/playground.html](https://simonw.github.io/justjshtml/playground.html)

![Image 1: Screenshot of JustJSHTML Playground web application. Header reads "JustJSHTML Playground" with subtitle "A dependency-free JavaScript HTML5 parser - GitHub". Below is a status bar showing "JavaScript Environment" with a green "Ready" badge. The main input area has "Paste HTML" and "Fetch from URL" buttons, with a text area containing HTML code: "<!DOCTYPE html> <html> <head> <title>Example Page</title> </head> <body> <header> <nav> <ul>". A "Playground Mode" section shows buttons for "CSS Selector Query", "Pretty Print HTML", "Tree Structure", "Stream Events", "Extract Text", and "To Markdown" (highlighted in purple). Below is a text field labeled "CSS Selector (optional - leave empty for whole document):" with placeholder "e.g., article, main, .content (or leave empty)" and a green "Convert to Markdown" button. The Output section has a teal header with "Whole document" badge and displays converted markdown: "Example Page" followed by "- [Home](/)" "- [About](/about)" "- [Contact](/contact)".](https://static.simonwillison.net/static/2025/justjshtml-playground.jpg)

All it needed now was some documentation:

> `Add a comprehensive README with full usage instructions including attribution plus how this was built plus how to use in in HTML plus how to use it in Node.js`

You can [read the result here](https://github.com/simonw/justjshtml/blob/f3a33fdb29bf97846fd017185edc8cf82783032e/README.md).

We are now at eight prompts total, running for just over four hours and I’ve decorated for Christmas and watched [Wake Up Dead Man](https://en.wikipedia.org/wiki/Wake_Up_Dead_Man) on Netflix.

According to Codex CLI:

> `Token usage: total=2,089,858 input=1,464,295 (+ 97,122,176 cached) output=625,563 (reasoning 437,010)`

My [llm-prices.com calculator](https://www.llm-prices.com/#it=2089858&cit=97122176&ot=625563&sel=gpt-5.2) estimates that at $29.41 if I was paying for those tokens at API prices, but they were included in my $20/month ChatGPT Plus subscription so the actual extra cost to me was zero.

#### What can we learn from this?[#](https://simonwillison.net/2025/Dec/15/porting-justhtml/#what-can-we-learn-from-this-)

I’m sharing this project because I think it demonstrates a bunch of interesting things about the state of LLMs in December 2025.

*   Frontier LLMs really can perform complex, multi-hour tasks with hundreds of tool calls and minimal supervision. I used GPT-5.2 for this but I have no reason to believe that Claude Opus 4.5 or Gemini 3 Pro would not be able to achieve the same thing—the only reason I haven’t tried is that I don’t want to burn another 4 hours of time and several million tokens on more runs.
*   If you can reduce a problem to a robust test suite you can set a coding agent loop loose on it with a high degree of confidence that it will eventually succeed. I called this [designing the agentic loop](https://simonwillison.net/2025/Sep/30/designing-agentic-loops/) a few months ago. I think it’s the key skill to unlocking the potential of LLMs for complex tasks.
*   Porting entire open source libraries from one language to another via a coding agent works extremely well.
*   Code is so cheap it’s practically free. Code that _works_ continues to carry a cost, but that cost has plummeted now that coding agents can check their work as they go.
*   We haven’t even _begun_ to unpack the etiquette and ethics around this style of development. Is it responsible and appropriate to churn out a direct port of a library like this in a few hours while watching a movie? What would it take for code built like this to be trusted in production?

I’ll end with some open questions:

*   Does this library represent a legal violation of copyright of either the Rust library or the Python one?
*   Even if this is legal, is it ethical to build a library in this way?
*   Does this format of development hurt the open source ecosystem?
*   Can I even assert copyright over this, given how much of the work was produced by the LLM?
*   Is it responsible to publish software libraries built in this way?
*   How much better would this library be if an expert team hand crafted it over the course of several months?