Title: My gift to the rustdoc team

URL Source: https://fasterthanli.me/articles/my-gift-to-the-rust-docs-team

Published Time: 2025-12-13T10:00:00Z

Markdown Content:
About two weeks ago I entered a discussion with the docs.rs team about, basically, why we have to look at this:

![Image 1: My browser showing a docs.rs page for a crate that I published myself, which contains a lot of different code blocks with different languages but they're all white on black. It's sad.
](https://cdn.fasterthanli.me/content/articles/my-gift-to-the-rust-docs-team/docsrs-no-colors@2x~eb49252c206ebe40.jxl)

When we could be looking at this:

![Image 2: My browser showing a docs.rs page for a crate that I published myself, which contains a lot of different code blocks with different languages. this time it's colored.
](https://cdn.fasterthanli.me/content/articles/my-gift-to-the-rust-docs-team/docsrs-yes-colors@2x~708d0f07e1265747.jxl)

And of course, as always, there are reasons why things are the way they are. In an effort to understand those reasons, I opened a GitHub issue which resulted in a [short but productive](https://github.com/rust-lang/docs.rs/issues/3040) discussion.

I walked away discouraged, and then decided to, reasons be damned, attack this problem from three different angles.

But first, the minimal required amount of background information on all this.

[Background ----------](https://fasterthanli.me/articles/my-gift-to-the-rust-docs-team#background)
Rust provides everyone with a tool that lets them generate HTML and JSON documentation for their crates, from [doc comments](https://doc.rust-lang.org/reference/comments.html#r-comments.doc) (`///`, or `//!` for modules).

Which is _amazing_. You can easily get offline documentation before hopping on a plane, and you can preview what your docs will look like before publication.

Once you’re done iterating on the documentation of your crate, which you should do because documentation is important, it’s time to publish your crate to [crates.io](https://crates.io/).

This puts your crate in the build queue at [docs.rs](https://docs.rs/), or rather, one of the two build queues, the one for nice people and the one for naughty people:

![Image 3: A screenshot of a browser window showing the docs.rs release queue. The currently being build section has two crates with names socket_port and gatio-der. And the build queue has a bunch of arborium crates with priority minus one. The two sections are labeled respectively "nice" and "naughty" with published 1200 crates this week. 
](https://cdn.fasterthanli.me/content/articles/my-gift-to-the-rust-docs-team/docsrs-queue@2x~4ed4f582c12cd749.jxl)

If/when the build succeeds, you get thrown [in the 7.75TiB bucket](https://github.com/rust-lang/docs.rs/issues/3040#issuecomment-3586845728) with the others and you get a little corner of the internet to call yours, with a fancy navbar that connects you to the right of the docs.rs-verse:

![Image 4: A screenshot of a documentation page as seen on the docs.rs website showing that they have a navigation bar on top added on the output of rustdoc
](https://cdn.fasterthanli.me/content/articles/my-gift-to-the-rust-docs-team/docsrs-navbar@2x~bfd2ccb682f8ba83.jxl)

The bucket contains a bunch of HTML, CSS, and JavaScript that is completely immutable, unless you run another rustdoc build from scratch (which the docs.rs team does for the latest version of all crates, but not historical versions).

This kind of explains the first reason why it is hard to just make those things colored. There is _no way in hell_ that we are rebuilding every version of every crate ever with the “I like colors” feature turned on. That’s simply not feasible.

[Problems --------](https://fasterthanli.me/articles/my-gift-to-the-rust-docs-team#problems)
And that’s just the first of many different problems.

First off, there are many different solutions to highlight code.

*   Which one do you pick?
*   Which languages do you include?
*   Can you trust it to run and to provide the quality output?
*   Does it require dynamic linking?
*   Does it build on all the target platforms that rustdoc supports?
*   The HTML markup for syntax highlighted code is bigger than for non-syntax highlighted code
    *   By how much?
    *   Can we even afford that?

*   Who’s gonna implement all this?

Well!

[tree-sitter](https://tree-sitter.github.io/tree-sitter/), 96 of them [by popular vote](https://bsky.app/profile/fasterthanli.me/post/3m6ugpxugtc2o), yes, no, yes, not much, probably, me.

[Solutions ---------](https://fasterthanli.me/articles/my-gift-to-the-rust-docs-team#solutions)
I have been using [tree-sitter](https://tree-sitter.github.io/tree-sitter/) for as long as I have over-engineered my website, which is six years now.

![Image 5: A screenshot of the TreeSitter website showing what it's good for. It says it's a parser generator tool, an incremental parsing library. It can build a concrete syntax tree for a source file and efficiently update the syntax tree as the source file is edited. The TreeSitter aims to be general, fast, robust, and dependency-free.
The rest is on the website if you want to go read it. 
](https://cdn.fasterthanli.me/content/articles/my-gift-to-the-rust-docs-team/tree-sitter@2x~b1805b56ed3f3530.jxl)

As far as I’m concerned, it is _the_ gold standard in terms of syntax highlighting that only an LSP can beat, but good luck convincing anyone to run that, to generate a bunch of documentation.

LSP meaning language server protocol, which is the language that [Rust Analyzer](https://rust-analyzer.github.io/) and your code editor speak. They are able to do semantic highlighting, but of course require loading all of your source code, all of its dependencies, and the entire sysroot, which takes a lot of time and memory.

Therefore, it is unsuitable for offline syntax highlighting. Well, I mean… don’t let me stop you. I’m a bear, not a cop.

However, even though there are crates for the [tree-sitter core](https://lib.rs/crates/tree-sitter) and for [tree-sitter-highlight](https://lib.rs/crates/tree-sitter-highlight), the rest you kind of have to put together yourself.

First, you have to find a grammar for your language. If your language is Rust or C++, then you’re in a very good position because a high quality grammar that’s up to date is available right now on the [tree-sitter-grammars GitHub org](https://github.com/tree-sitter-grammars).

But if your tastes are a little more uncommon, then you might find yourself searching for the perfect grammar for quite some time, or even writing your own.

Or, finding one that looks like it might be okay but was actually written against a much older version of tree-sitter and needs to be cleaned up and regenerated, with some weird rules removed because they make the compilation time explode…

“regenerate” in this context means taking the grammar.js and possibly scanner.cc of the grammar repository and rerunning it through the tree-sitter CLI, which is going to generate a mountain of C code for the actual parser.

You have to do that, of course, for every language you want to highlight:

![Image 6: A screenshot of the tree-sitter collection repository, which is closed
source, and includes bash, C, Clojure, Dockerfile, Go, HTML, INI, Java,
JavaScript, Meson, Nix, Python, Rust, TOML, TypeScript, x86asm, YAML,
and Zig — 18 different grammars. 
](https://cdn.fasterthanli.me/content/articles/my-gift-to-the-rust-docs-team/tree-sitter-collection@2x~4d562b60c944a06d.jxl)

I collected 18 different grammars before I started wondering if I couldn’t solve the problem for everyone once and for all, especially since I started having different projects that all needed to highlight something.

What those grammars and the automatically generated crate alongside them do is export a single symbol, which is a pointer to a struct that contains parsing tables along with function pointers to the scanner if there’s one, etc.

![Image 7: A capture of the docs.rs page for tree-sitter-rust, showing that the only export, besides some queries, is the language function, which is a pointer to a struct. 
](https://cdn.fasterthanli.me/content/articles/my-gift-to-the-rust-docs-team/tree-sitter-rust-exports@2x~1c72a1c21600b82c.jxl)

It is not ready to use by any stretch of the imagination.

Actually, I lied, and you can see it on that screenshot. It exports other things if you’re lucky, like highlights query and injections query, which you need if you want to actually highlight the result of parsing code into a tree.

If you don’t have highlights queries, then you have a tree of nodes, but you don’t know which corresponds to what. You don’t know what’s a keyword, what’s a function, what’s a number, a string, anything that could have some sort of meaningful color.

![Image 8: A screenshot of the WebAssembly playground you can get when you compile a tree sitter grammar. It shows the code that's being parsed top left, the tree on the right, and bottom left we have queries that are used to highlight. 
](https://cdn.fasterthanli.me/content/articles/my-gift-to-the-rust-docs-team/wasm-playground@2x~e8ffbb75c76708d6.jxl)

You don’t know how to match your color theme to all the nodes that you have. That’s what the highlights query does. As for the injections queries, they let you know what other grammar is nested inside of yours.

For example, Svelte components typically are HTML and can embed scripts and styles. So you inject JavaScript and CSS in there, and sometimes TypeScript.

There is a callback system in tree-sitter-highlight to handle injections, but having the right dependencies and implementing that callback are all up to you!

Unless you’re me and you’ve been dealing with that problem for 6 years and you have your own private stash of all the good grammars.

That changes today: I am happy to announce: [arborium](https://arborium.bearcove.eu/).

![Image 9: A screenshot of the Arborium homepage showing a Rust code sample. It includes links, a language picker, and a theme picker with light and dark themes. There's also a randomize button and a little tidbit about every programming language featured on the right.  ](https://cdn.fasterthanli.me/content/articles/my-gift-to-the-rust-docs-team/arborium-homepage@2x~a4c5d3c6335a4953.jxl)

[arborium home apge](https://arborium.bearcove.eu/)

[arborium --------](https://fasterthanli.me/articles/my-gift-to-the-rust-docs-team#arborium)
For the 96 languages that people requested, I have gone and searched for the best available grammar, and I have vendored it, fixed it up, made sure the highlight queries worked, made sure the license and attribution are present in my redistribution of them, and integrated it into one of the cargo feature flags of the [main arborium crate](https://docs.rs/arborium).

![Image 10: A screenshot of the Arborium 1.3.0 release showing a bunch of feature flags. They don't all fit on the page...
](https://cdn.fasterthanli.me/content/articles/my-gift-to-the-rust-docs-team/arborium-docs@2x~0f8da0249b14c477.jxl)

But it goes a little further. If you depend, for example, on Svelte, then it’s also going to bring the crates that are needed to highlight the Svelte component fully, namely HTML, CSS, and JavaScript.

![Image 11: Screenshot of arborium-svelte's dependencies, which includes arborium-html, arborium-css, arborium-jaascript, arborium-scss etc.
](https://cdn.fasterthanli.me/content/articles/my-gift-to-the-rust-docs-team/svelte-dependencies@2x~3003eb63ca1bb6a8.jxl)

Much like the original tree-sitter crates, they cannot actually do much by themselves, but you’re supposed to use them through the main Arborium crate, which has very simple interfaces to highlight code:

```
use arborium::Highlighter;

let mut highlighter = Highlighter::new();
let html = highlighter.highlight_to_html("rust", "fn main() {}")?;
```

Granted, here we are kind of eschewing the subtlety of incremental parsing and highlighting that tree-sitter provides, but don’t worry, there are more complicated APIs right there if you need them.

Everything can be configured from the theme, of which we ship a fair amount built in, to the style of the HTML output, by default we go for the modern, compact, and widely-supported:

```
<a-k>keyword</a-k>
```

If you insist on being retro and pinky promise that Brotli compression makes up for it anyway, then you can use the long-winded alternative:

```
<span class="code-keyword">keyword</span>
```

If you’re more of a terminal kind of person, then you can have its output and see escapes. Even with an optional background color, some margin and padding, and a border, if you really want to make it stand out:

![Image 12: A screenshot of my terminal showing some Rust code highlighted with the Tokyo night theme, some Haskell code highlighted with the Kanagawa dragon theme, and some Svelte code highlighted with the rosé pine moon theme. 
](https://cdn.fasterthanli.me/content/articles/my-gift-to-the-rust-docs-team/cleanshot-2025-12-13-at-17.12.38@2x~527bdcaf6c69628b.jxl)

And perhaps most importantly, the rust crates are set up in such a way that they can compile through cargo to the `wasm32-unknown-unknown` target.

This was the thing that tripped me up because it requires providing just enough libc symbols so that the grammars are happy.


```
crates/arborium-sysroot/wasm-sysroot › main 󰏗 1󰏫 18via  v17.0.0-clang  › 18:10 🪴
› ls --tree
.
├── assert.h
├── ctype.h
├── endian.h
├── inttypes.h
(cut)
```

![Image 13: Cool bear](https://cdn.fasterthanli.me/content/img/reimena/coolbear-neutral~7010eabf1d70d303.jxl)

But Amos! Didn’t you just show a “WASM playground” that you got by running `tree-sitter build --wasm` then `tree-sitter playground`?

Yeah, they target `wasm32-wasi`

Well, that’s because they build for [`wasm32-wasi`](https://github.com/tree-sitter/tree-sitter/blob/98de2bc1a87bd2e7ef7f299fbd8843400978efe4/crates/loader/src/loader.rs#L1301-L1341), which is _slightly different_. At the end of the day, someone has to provide system functions, and in our case, it’s me.

Most functions provided are simple (`isupper`, `islower`) etc., with the exception of `malloc`, `free` and friends, which in arborium’s case, are provided by [`dlmalloc`](https://lib.rs/crates/dlmalloc).

Because all of those crates compile with a Rust toolchain (that invokes a C toolchain) to `wasm32-unknown-unknown`, we can run them in a browser. With a little glue!

[Angle 1: just include this script ---------------------------------](https://fasterthanli.me/articles/my-gift-to-the-rust-docs-team#angle-1-just-include-this-script)
Right now, if you publish a crate and want the documentation to be highlighted for languages other than Rust, you can follow the instructions at [arborium.bearcove.eu](https://arborium.bearcove.eu/), to:

*   Create an HTML file within your repository
*   Add metadata to your Cargo.toml file so the docs.rs build process picks it up

You can see this in action on the [arboriu_docsrs_demo page](https://docs.rs/arborium-docsrs-demo/), and its sources in the [arborium repository](https://github.com/bearcove/arborium/tree/main/crates/arborium-docsrs-demo)

Your browser does not support the video tag.

I even went the little extra mile of detecting that you’re running on docs.rs and matching the theme that is currently active in a responsive way. So it’s gonna use docs.rs light, docs.rs dark, and the Ayu theme, depending on whatever the page does.

![Image 14: Amos](https://cdn.fasterthanli.me/content/img/reimena/amos-neutral~55a3477398fe0cb1.jxl)

Those themes do not appeal to my personal aesthetic, but I decided that consistency was the most important imperative here.

This solution is great because it works today.

It’s great because it means zero extra work for the Rust docs team. They don’t have to mess with Rustdoc, their build pipeline, or their infrastructure. It just works. It’s a wonderful escape hatch.

People have used it to integrate KaTeX (render LaTeX equations), to render diagrams, and do all sorts of things on the front-end.

![Image 15: A screenshot of a rustdoc katex demo ](https://cdn.fasterthanli.me/content/articles/my-gift-to-the-rust-docs-team/katex-example@2x~71bfc88980316b26.jxl)

[rustdoc-katex-demo](https://docs.rs/rustdoc-katex-demo/latest/rustdoc_katex_demo/)

This solution is _also_ the worst! Because it requires not just JavaScript but also WebAssembly, it forces people to download large grammar bundles (sometimes hundreds of kilobytes!) just to highlight small code blocks.

But most importantly, it’s a security disaster waiting to happen.

You should never let anyone inject third-party JavaScript into the main context of your page. Right now on docs.rs, there’s not much to steal except your favorite theme, but that might not always be the case. It’s just bad practice, and the team knows it—they want, or should want, to close that hole.

If you’re confused about why this is so bad, imagine everyone adopts Arborium as the main way of highlighting code on their docs.rs pages. A few years down the line, I decide to turn evil. All I have to do is publish a malicious version of the arborium package on NPM to reach millions of people instantly.

![Image 16: A team of hackers wearing hoodies ](https://cdn.fasterthanli.me/content/articles/my-gift-to-the-rust-docs-team/hackers~30b8696bababf38f.jxl)

Contrary to popular belief and this stock photo I paid a monthly subscription for and I'm DAMN WELL gonna use, you don't _need_ to wear a hoodie to do hacking.

You could, of course, have people pin to a specific version of the Arborium package, but that would also prevent them from getting important updates. Ideally, all the JavaScript distributed on docs.rs pages should come from the docs team, so that the world is only in danger if the docs teams themselves turn evil.

Therefore, in the long term, in a world where we have money and people and time to address this, we must consider two other angles.

[Angle 2: it goes in the rustdoc hole ------------------------------------](https://fasterthanli.me/articles/my-gift-to-the-rust-docs-team#angle-2-it-goes-in-the-rustdoc-hole)
Arborium is just a bunch of Rust crates that contains a bunch of C code, both of which are extremely portable. There is nothing funky going on here, there is no dynamic linking, there is no plugin folder, asynchronous loading or whatever. Just a bunch of grammars and code that you need to actually highlight things.

Therefore, I was able to make a PR against RustDoc to get it to highlight other languages:

![Image 17: A screenshot of the PR in question ](https://cdn.fasterthanli.me/content/articles/my-gift-to-the-rust-docs-team/rustdoc-pr@2x~61532216ab9b7c22.jxl)

[rust PR #149944](https://github.com/rust-lang/rust/pull/149944)

At +537-11, it’s a pretty small PR, that in reality pulls literal millions of lines of C code (parsers generated by tree-sitter).

This makes the question of “what grammars do we bundle?” all the more important—thankfully, I’m not going to be the one who solves it.


```
rust › rustdoc-arborium 󰏫 3via  v3.14.2  › 00:54 🪴
› ls -lhA build/aarch64-apple-darwin/stage2/bin/rustdoc
Permissions Size User Date Modified Name
.rwxr-xr-x  171M amos 14 Dec 00:52  build/aarch64-apple-darwin/stage2/bin/rustdoc
```

```
rust › main via  v3.14.2  › 01:44 🪴
› ls -lhA build/aarch64-apple-darwin/stage2/bin/rustdoc
Permissions Size User Date Modified Name
.rwxr-xr-x   22M amos 14 Dec 01:44  build/aarch64-apple-darwin/stage2/bin/rustdoc
```

![Image 18: Amos](https://cdn.fasterthanli.me/content/img/reimena/amos-neutral~55a3477398fe0cb1.jxl)

Top: a custom rustdoc with all 96 languages compiled in. Bottom: “main branch” rustdoc.

I fully anticipate that at some point in the discussion someone might look at those binary sizes and go: “yeesh, I don’t think we can do that”.

Consequently, I present to you: angle number three.

[Angle 3: only in the backend ----------------------------](https://fasterthanli.me/articles/my-gift-to-the-rust-docs-team#angle-3-only-in-the-backend)
If it’s not feasible to afford everyone the luxury of highlighting hundreds of programming, markup, and configuration languages at home, then I will settle for doing the deed in the backend of docs.rs.

Enter: [arborium-rustdoc](https://github.com/bearcove/arborium/tree/main/crates/arborium-rustdoc).

It’s a post-processor specifically for rustdoc. It detects code blocks in HTML files and highlights them! It also patches the main CSS file to add its styles at the bottom.

I tested it on _all dependencies of the [facet](https://github.com/facet-rs/facet) monorepo_, and the size of the ~900MB doc folder went up by a whopping 24KB!

Your browser does not support the video tag.

I really hope we can afford this. I’m even willing to personally chip in.

[Post-mortem -----------](https://fasterthanli.me/articles/my-gift-to-the-rust-docs-team#post-mortem)
The most challenging part of this whole project was probably the CI set up: when building a small package, GitHub Actions is bearable. When orchestrating 2x96 builds + supporting packages and publishing with provenance to two platforms, it really isn’t.

I’d like to thank [Depot.dev](https://depot.dev/) for generously donating their beefy CI runners, without which I would’ve just bailed out of this project early.

Even then, I distributed plugin jobs into ten tree-themed groups:

![Image 19: A graph of all the github actions jobs that go into publishing arborium.
](https://cdn.fasterthanli.me/content/articles/my-gift-to-the-rust-docs-team/github-actions-graph@2x~ec68f7f05eceaf0b.jxl)

Any CI failure is _punishing_, so I kept as much of the logic as possible out of YAML, and into a [cargo-xtask](https://github.com/matklad/cargo-xtask). It’s actually very friendly!

Your browser does not support the video tag.

But it’s not just progress bars and nerd font icons. It’s also making sure that every single artifact we produce can be loaded in a browser by parsing the WebAssembly bundle and checking its imports, via [walrus](https://lib.rs/crates/walrus) (instead of summarily piping `wasm-objdump -x` into grep or whatever).

There’s _a lot_ of build engineering going on here. I’m using blake3 hashes to avoid recomputing inputs, mostly because I think the name sounds cool, a dozen crazy things happened during those two weeks and I barely remember the half of it.

[Conclusion ----------](https://fasterthanli.me/articles/my-gift-to-the-rust-docs-team#conclusion)
I built arborium so it could last us for the next 20 years. I’m thrilled to donate it to the commons (it’s Apache2+MIT) and to, hopefully, see _accurate_ syntax highlighting blossom on the web, just like we’ve seen code editors suddenly get better at it before.

I believe tree-sitter can change the world _a second time_. This time, for everyone who simply doesn’t have the time or know-how to put all the pieces together.

All the details are on the [arborium website](https://arborium.bearcove.eu/).

![Image 20: Amos](https://cdn.fasterthanli.me/content/img/reimena/amos-neutral~55a3477398fe0cb1.jxl)

For docs.rs specifically, if I had to do it, realistically? I’d go with [arborium-rustdoc](https://github.com/bearcove/arborium/tree/main/crates/arborium-rustdoc) as a post-processing step. It’s fast, you can build it with support for _all_ languages, and it doesn’t have any of the security or bundle size implications of the other two solutions. You can even sandbox it!

Happy holidays!

Thanks to my sponsors:

### Help us test our Github/Patreon/Discord integration

If you can, consider supporting this work at a tier you can afford:

Bronze Tier*    Access to bonus content (Rust codebases etc.)
*    Access to private Discord channels
Silver Tier*   All perks from other tiers, plus:
*    Early access to videos and articles
*    Name credited in videos and articles
Gold Tier*   All perks from other tiers, plus:
*    My eternal gratitude

Did you know I also make videos? Check them out on [PeerTube](https://tube.fasterthanli.me/) and also [YouTube](https://youtube.com/@fasterthanlime)!

Here's another article just for you:

[My ideal Rust workflow](https://fasterthanli.me/articles/my-ideal-rust-workflow)
---------------------------------------------------------------------------------

[Writing](https://fasterthanli.me/articles/my-ideal-rust-workflow)[Rust](https://www.rust-lang.org/) is pretty neat. But you know what’s even neater? Continuously testing Rust, releasing Rust, and eventually, shipping Rust to production. And for that, we want more than plug-in for a code editor.

We want… a workflow.

[Why I specifically care about this ----------------------------------](https://fasterthanli.me/articles/my-gift-to-the-rust-docs-team#why-i-specifically-care-about-this)

This gets pretty long, so if all you want is the advice, feel free to [jump to it](https://fasterthanli.me/articles/my-gift-to-the-rust-docs-team#advice-start) directly.