Title: Thoughts on Go vs. Rust vs. Zig

URL Source: https://sinclairtarget.com/blog/2025/08/thoughts-on-go-vs.-rust-vs.-zig/

Markdown Content:
#### Aug 09, 2025

I realized recently that rather than using “the right tool for the job” I’ve been using the tool _at_ the job and that’s mostly determined the programming languages I know. So over the last couple months I’ve put a lot of time into experimenting with languages I don’t get to use at work. My goal hasn’t been proficiency; I’m more interested in forming an opinion on what each language is good for.

Programming languages differ along so many axes that it can be hard to compare them without defaulting to the obviously true but 1) entirely boring and 2) not-that-helpful conclusion that there are trade-offs. Of course there are trade-offs. The important question is, why did this language commit to this particular set of trade-offs?

That question is interesting to me because I don’t want to choose a language based on a list of features as if I were buying a humidifier. I care about building software and I care about my tools. In making the trade-offs they make, languages express a set of values. I’d like to find out which values resonate with me.

That question is also useful in clarifying the difference between languages that, at the end of the day, have feature sets that significantly overlap. If the number of questions online about “Go vs. Rust” or “Rust vs. Zig” is a reliable metric, people are confused. It’s hard to remember, say, that language _X_ is better for writing web services because it has features _a_, _b_, and _c_ whereas language _Y_ only has features _a_ and _b_. Easier, I think, to remember that language _X_ is better for writing web services because language _Y_ was designed by someone who hates the internet (let’s imagine) and believes we should unplug the whole thing.

I’ve collected here my impressions of the three languages I’ve experimented with lately: Go, Rust, and Zig. I’ve tried to synthesize my experience with each language into a sweeping verdict on what that language values and how well it executes on those values. This might be reductive, but, like, crystallizing a set of reductive prejudices is sort of what I’m trying to do here.

Go
--

Go is distinguished by its minimalism. It has been described as “a modern C.” Go isn’t like C, because it is garbage-collected and has a real run-time, but it _is_ like C in that you can fit the whole language in your head.

You can fit the whole language in your head because Go has so few features. For a long time, Go was notorious for not having generics. That was finally changed in Go 1.18, but that was only after 12 years of people begging for generics to be added to the language. Other features common in modern languages, like tagged unions or syntactic sugar for error-handling, have not been added to Go.

It seems the Go development team has a high bar for adding features to the language. The end result is a language that forces you to write a lot of boilerplate code to implement logic that could be more succinctly expressed in another language. But the result is also a language that is stable over time and easy to read.

To give you another example of Go’s minimalism, consider Go’s slice type. Both Rust and Zig have a slice type, but these are fat pointers and fat pointers only. In Go, a slice is a fat pointer to a contiguous sequence in memory, but a slice can also grow, meaning that it subsumes the functionality of Rust’s `Vec<T>` type and Zig’s `ArrayList`. Also, since Go is managing your memory for you, Go will decide whether your slice’s backing memory lives on the stack or the heap; in Rust or Zig, you have to think much harder about where your memory lives.

Go’s origin myth, as I understand it, is basically this: Rob Pike was sick of waiting for C++ projects to compile and was sick of other programmers at Google making mistakes in those same C++ projects. Go is therefore simple where C++ is baroque. It is a language for the programming rank and file, designed to be sufficient for 90% of use cases while also being easy to understand, even (perhaps especially) when writing concurrent code.

I don’t use Go at work, but I think I should. Go is minimal in service of corporate collaboration. I don’t mean that as a slight—building software in a corporate environment has its own challenges, which Go solves for.

Rust
----

Where Go is minimalist, Rust is maximalist. A tagline often associated with Rust is “zero-cost abstractions.” I would amend that to read, “zero-cost abstractions, and lots of them!”

Rust has a reputation for being hard to learn. I agree with Jamie Brandon, who writes that [it’s not lifetimes that make Rust difficult](https://www.scattered-thoughts.net/writing/assorted-thoughts-on-zig-and-rust/), it’s the number of concepts stuffed into the language. I’m not the first person to pick on [this particular Github comment](https://github.com/rust-lang/rust/issues/68015#issuecomment-835786438), but it perfectly illustrates the conceptual density of Rust:

> The type `Pin<&LocalType>` implements `Deref<Target = LocalType>` but it doesn’t implement `DerefMut`. The types `Pin` and `&` are `#[fundamental]` so that an `impl DerefMut` for `Pin<&LocalType>>` is possible. You can use `LocalType == SomeLocalStruct` or `LocalType == dyn LocalTrait` and you can coerce `Pin<Pin<&SomeLocalStruct>>` into `Pin<Pin<&dyn LocalTrait>>`. (Indeed, two layers of Pin!!) This allows creating a pair of “smart pointers that implement `CoerceUnsized` but have strange behavior” on stable (`Pin<&SomeLocalStruct>` and `Pin<&dyn LocalTrait>` become the smart pointers with “strange behavior” and they already implement `CoerceUnsized`).

Of course, Rust isn’t trying to be maximalist the same way Go is trying to be minimalist. Rust is a complex language because what it’s trying to do is deliver on two goals—safety and performance—that are somewhat in tension.

The performance goal is self-explanatory. What “safety” means is less clear; at least it was to me, though maybe I’ve just been Python-brained for too long. “Safety” means “memory safety,” the idea that you shouldn’t be able to dereference an invalid pointer, or do a double-free, etc. But it also means more than that. A “safe” program avoids all undefined behavior (sometimes referred to as “UB”).

What is the dreaded UB? I think the best way to understand it is to remember that, for any running program, there are FATES WORSE THAN DEATH. If something goes wrong in your program, immediate termination is great actually! Because the alternative, if the error isn’t caught, is that your program crosses over into a twilight zone of unpredictability, where its behavior might be determined by which thread wins the next data race or by what garbage happens to be at a particular memory address. Now you have heisenbugs and security vulnerabilities. Very bad.

Rust tries to prevent UB without paying any run-time performance penalty by checking for it at compile-time. The Rust compiler is smart, but it’s not omniscient. For it to be able to check your code, it has to understand what your code will do at run-time. And so Rust has an expressive type system and a menagerie of traits that allow you to express, to the compiler, what in another language would just be the apparent run-time behavior of your code.

This makes Rust hard, because you can’t just _do_ the thing! You have to find out Rust’s name for the thing—find the trait or whatever you need—then implement it as Rust expects you to. But if you do this, Rust can make guarantees about the behavior of your code that other languages cannot, which depending on your application might be crucial. It can also make guarantees about _other people’s_ code, which makes consuming libraries easy in Rust and explains why Rust projects have almost as many dependencies as projects in the JavaScript ecosystem.

Zig
---

Of the three languages, Zig is the newest and least mature. As of this writing, Zig is only on version 0.14. Its standard library has almost zero documentation and the best way to learn how to use it is to consult the source code directly.

Although I don’t know if this is true, I like to think of Zig as a reaction to both Go and Rust. Go is simple because it obscures details about how the computer actually works. Rust is safe because it forces you to jump through its many hoops. Zig will set you free! In Zig, you control the universe and nobody can tell you what to do.

In both Go and Rust, allocating an object on the heap is as easy as returning a pointer to a struct from a function. The allocation is implicit. In Zig, you allocate every byte yourself, explicitly. (Zig has manual memory management.) You have more control here than you have even in C: To allocate bytes, you have to call `alloc()` on a specific kind of allocator, meaning you have to decide on the best allocator implementation for your use case.

In Rust, creating a mutable global variable is so hard that there are [long forum discussions](https://users.rust-lang.org/t/whats-the-correct-way-of-doing-static-mut-in-2024-rust/120403) on how to do it. In Zig, you can just create one, no problem.

Undefined behavior is still important in Zig. Zig calls it “illegal behavior.” It tries to detect it at run-time and crash the program when it occurs. For those who might worry about the performance cost of these checks, Zig offers four different “release modes” that you can choose from when you build your program. In some of these, the checks are disabled. The idea seems to be that you can run your program enough times in the checked release modes to have reasonable confidence that there will be no illegal behavior in the unchecked build of your program. That seems like a highly pragmatic design to me.

Another difference between Zig and the other two languages is Zig’s relationship to object-oriented programming. OOP has been out of favor for a while now and both Go and Rust eschew class inheritance. But Go and Rust have enough support for other object-oriented programming idioms that you could still construct your program as a graph of interacting objects if you wanted to. Zig has methods, but no private struct fields and no language feature implementing run-time polymorphism (AKA dynamic dispatch), even though `std.mem.Allocator` is _dying_ to be an interface. As best as I can tell, these exclusions are intentional; Zig is a language for [data-oriented design](https://www.youtube.com/watch?v=IroPQ150F6c).

One more thing I want to say about this, because I found it eye-opening: It might seem crazy to be building a programming language with manual memory management in 2025, especially when Rust has shown that you don’t even need garbage collection and can let the compiler do it for you. But this is a design choice very much related to the choice to exclude OOP features. In Go and Rust and so many other languages, you tend to allocate little bits of memory at a time for each object in your object graph. Your program has thousands of little hidden `malloc()`s and `free()`s, and therefore thousands of different lifetimes. [This is RAII](https://www.youtube.com/watch?v=xt1KNDmOYqA). In Zig, it might seem like manual memory management would require lots of tedious, error-prone bookkeeping, but that’s only if you insist on tying memory allocations to all your little objects. You could instead just allocate and free big chunks of memory at certain sensible points in your program (like at the start of each iteration of your event loop), and use that memory to hold the data you need to operate on. It’s this approach that Zig encourages.

Many people [seem confused](https://www.reddit.com/r/rust/comments/1333zs1/zig_or_rust/) about why Zig should exist if Rust does already. It’s not just that Zig is trying to be simpler. I think this difference is the more important one. Zig wants you to excise even more object-oriented thinking from your code.

Zig has a fun, subversive feel to it. It’s a language for smashing the corporate class hierarchy (of objects). It’s a language for megalomaniacs and anarchists. I like it. I hope it gets to a stable release soon, though the Zig team’s current priority seems to be [rewriting all of their dependencies](https://github.com/ziglang/zig/issues/16270). It’s not impossible they try to rewrite the Linux kernel before we see Zig 1.0.