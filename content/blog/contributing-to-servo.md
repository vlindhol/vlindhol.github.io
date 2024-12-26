---
title: "My experience contributing to Servo"
date: 2024-12-09T12:14:57+03:00
tags: ["programming", "rust"]
draft: true
---

I've been a long-time Rust enthusiast, but haven't worked a lot with Rust in a real way. Thus, as I found myself with some spare time during my sabbatical, I decided to find an open source Rust project to contribute to, and Servo fit that bill perfectly.

Why Servo? It's a browser engine written entirely in Rust, started at Mozilla, where Rust was invented. The stated (and unstated) goals tick a lot of boxes for me:

* The browser landscape is practically a Chrome monoculture, I want to help counteract that.
* The future of UI for most apps will either be a more polished PWA experience (which depends on OS maker support) or less memory and processor intensive desktop web apps á la Electron (which doesn't). Servo is designed to be embeddable, i.e. "used as a library", so it fits nicely into either future.
* Compared to incumbents, Servo is designed for GPUs and parallel processing from the ground up. It's state of the art in many ways.
* I like building web apps, so this gives me a deeper appreciation of how the DOM and rendering really works.

## Orienting myself in the project

As a start, I headed straight to the [Servo Github repo](https://github.com/servo/servo/). I quickly found myself reading [the Servo Book](https://book.servo.org/), which is meant to serve as the one-stop shop for documentation on using/hacking Servo.

I'll be quick to point out that while there is some useful info in the book, it definitely doesn't feel complete nor polished. Information is also found in the [wiki at the Github repo](https://github.com/servo/servo/wiki), with many pages currently only containing "this content has moved" links to the book, while other pages like the [2025 roadmap](https://github.com/servo/servo/wiki/Roadmap) fits in there naturally.

Last not but least I encourage would-be contributors to join the [Servo Zulip chat](https://servo.zulipchat.com/), which is a great place to ask for guidance and direction.

## My contribution: an XPath parser and evaluator

My strategy for finding something to work on wasn't very sophisticated. I spent a few days looking at newly-opened Issues in Github, fixing some truly miniscule bugs and linter warnings. After a while I happened upon an issue regarding [htmx not working](https://github.com/servo/servo/issues/34060), due to XPath queries not being implemented at all.

This seemed perfect: a tightly defined, reasonably substantial piece of work that nobody else would probably deem sexy enough to want to pick up any time soon, leaving me plenty of time to experiment and learn. Besides, I have a soft spot for `htmx`, a library I've reached for a few times when wanting to keep a website light on the JS. I want Servo to support it!

### The story of an XPath implementation

This project was as much about doing Rust "properly" as it was implementing the feature. I started out looking at crates that would do most of the work for me, but turns out that XML and XPath isn't super hot in 2024 🙈. There were some options available for parsing and evaluating XPath, but the implementations all wanted to parse the XML themselves and traverse the node tree using their own datastructures. Servo obviously has its own DOM Node implementation, and I want to use that since it's already in memory.

Last but not least, the XPath 1.0 standard (there's also 2.0 and 3.0 available) is for pure XML, whereas "XPath for DOM" lives in a slightly murky grey area that is about 95% XPath 1.0 and 5% "how browsers actually do it". Turns out that there's a need for someone to write down the actual spec!

At this point, however, I was more interested in getting some results, so I soon decided to write my own parser and evaluator, right in the Servo repo.

Step one was the parser, which could almost be a separate crate in and of itself, since it's all about converting a string into an AST tree. Maybe one day it'll be a crate, but for now it's just a single `parser.rs` file. I've used [nom](https://github.com/rust-bakery/nom) to great effect before, so that's what I ended up using here as well. In the long term it would be great to make a hand-written parser with really good error messages, but this was a good enough start. Luckily the XPath 1.0 grammar is pretty clearly defined, and is exactly the same in the DOM, so this part was a bit bothersome but not too challenging.

Step two was the evaluator, which takes the AST produced by the parser in step one, and a "starting node" (called the *context node*) from the DOM tree, and produces a valid `XPathResult`, which is either a primitive value or (more commonly) a set of matching DOM nodes.

At this point I really needed to get my hands dirty with Servo.

### Adding DOM features in Servo: the moving parts

Servo is a Cargo workspace, with the gargantuan `script` module being where all my work needed to take place. This is where everything DOM and JavaScript-related happens. The main things I ended up needing to learn was:

1. How memory management works (spoiler: the JS runtime does garbage collection)
2. WebIDL
3. Running WPT tests effectively

**Memory management**, which can get tricky in any Rust project, turned out to be quite painless (from my point of view). There's a [detailed writeup](XXX) about it, but the gist of it is:

1. All structs that correspond to classes in the DOM spec (e.g. `Element`) are marked with `#[dom_struct]`, which among other things is an alias for `#[derive(JSTraceable)]`.
2. Once the JS runtime (called SpiderMonkey) is aware of the existence of such a struct, its garbage collector looks at every field, looking for other `JSTraceable` contents. This type of GC is literally called [tracing garbage collection](XXX).
3. Once SpiderMonkey can't trace a `JSTraceable` object, the memory can safely get freed by the garbage collector *i.e. not by Rust logic*.
4. Not every `JSTraceable` struct lives inside another `JSTraceable`, so you can create a new GC "root" by saying `DomRoot::new(&ref_to_a_dom_struct)`. A root is simply one of many "starting points" for the GC's tracing.

From a Rustacean's perspective this means that for complex tree structures like the DOM, one doesn't need to think quite so much about lifetimes, since one rarely stores plain references to anything, put rather an opaque "this value is part of the DOM and thus gets GC'd" type. Like so:

```rust
// "traditional" Rust
struct Node {
    parent_node: Option<&Node>,
}

// GC'd by SpiderMonkey
#[dom_struct]
struct Node {
    parent_node: Dom<Option<Node>>
}
```

`Dom<T>` implement `Deref`, so you can still get ahold of a `&Node` by saying `&some_node.parent_node` etc. And you can at any time safely convert a `&Node` back to a traced `Dom<Node>` with `DomRoot::from_ref(some_node_ref)` (`DomRoot<T>` is a `Root<Dom<T>>`, the `Root` signifies that a new root has been stored for GC).

Are there any drawbacks? The main thing I've noted so far is that there are a lot of roots created "just in case", since it can be quite difficult to determine whether a root should be created or not. Picture a simple document with no JS scripting: optimally there would be only ONE root, which would be the containing `Document`, and all DOM objects can be found through that. But as soon as any type of JS gets involved, it's possible that the same object is reachable from many roots, which makes the garbage collector "double count" the same object. This is harmless, because all we care about is that the count is >0, but it can get wasteful.

As a fairly new Rustacean, I was thankful for the option of being able to ignore lifetimes, since an XPath evaluator traverses the DOM tree quite intensely. At first I tried using only pure `&Node` references, but it quickly got messy. Someone more seasoned could probably optimize the code, but now I ended up creating a lot of extraneous roots for simplicity of the implementation.

