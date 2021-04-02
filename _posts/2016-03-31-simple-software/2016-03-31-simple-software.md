---
layout: post
title: "Simple software"
date: "2016-03-31"
permalink: /blog/post/simple-software
---

Now that the dust has settled from [Square's acquisition of Framed](http://techcrunch.com/2016/03/14/square-brings-on-the-team-behind-framed-data-a-predictive-analytics-startup/), I wanted to take some time to talk about the building blocks of simple software, and the things I think we got right at Framed.

The Framed platform was written entirely in Clojure, and at its peak it was processing hundreds of millions of analytics events per day and running machine learning models over tens of millions of input points with thousands of features. <break />Despite being a full-fledged production-grade data platform, a number of techniques enabled us to keep the codebase refreshingly simple and easy to work in. I think a lot of the complexity we face every day in software is incidental (as opposed to inherent to the particular domain), and can be reined in.

The examples and arguments in this post will mostly be contrasting Clojure and Ruby as those make up the bulk of my professional experience, although the point is not really to argue the specifics of either or write an ode to Clojure. Instead, this is really about doing more with less. I don't expect many people to be intimately familiar with Clojure, so I'll try and explain things as I go.

## Namespaces / functions
In Clojure, code is organized using namespaces (`ns`) which contain functions, and that's pretty much it. Namespaces are explicitly imported in code that requires them, and functions or definitions within them are referred to with a qualified name.

A trivial example:

<pre class="prettyprint lang-clojure"><code>(ns myapp.calculator)

(defn add [x y]
  (+ x y))</code></pre>

<pre class="prettyprint lang-clojure"><code>(ns myapp.core
 (:require [myapp.calculator :as calc]))

(defn do-important-work
  (let [x (rand 100)
        y (rand 100)
        sum (calc/add x y)]
    (process-result sum)))</code></pre>

Here, `myapp.core` loads our `calculator` namespace and chooses to call it `calc`; from then on any functions within are explicitly referred to, as in `calc/add` and so on. All that's required for this is some extremely basic classpath config in the top-level project file (saying that Clojure code is in the `src` directory, for example), and then the problem is basically solved across our entire project. `foo/bar/quux.clj` defines `(ns foo.bar.quux)` and is imported as such; all external dependencies are also explicitly imported and used. This is in stark contrast to Ruby's `LOAD_PATH` modification, Rails' magical autoloading, disgusting `../../../` guesswork with `require_relative` and all the rest.

<p class="subtle">I have to mention here that it's technically possible to dodge the explicit namespace references in Clojure with things like `use` and `refer`. There is virtually never a good reason to do this. Clojure namespaces do have their own flaws, but they're pretty damn good.</p>

Using explicitly-specified dependencies and namespace/package names is admittedly just an opinion of mine rather than an objectively-better approach; modern IDEs/ctags/etc make jumping around codebases and locating things very feasible, but honestly doing things this way in Clojure reduced mental overhead to zero and was a total joy. I'd develop all my software this way if I could. Surprisingly, I can think of very few modern languages that enable this workflow. Haskell does if you choose to `import qualified` everything, although that doesn't seem to be the widespread approach. In Scala/Java/Ruby, about as close as you can get is a container object with a bunch of static methods, e.g. `Calculator.add(x, y)`, which is an okay approximation at best.

It's admittedly very easy to think in terms of OOP, and view this approach as almost primitive. I used to think to myself "well, in a *real* app you'd need X", where X is classes or a complicated inheritance hierarchy or design patterns and the like. However it's *shocking* the level of sophistication one can achieve with these simple building blocks alone. Clojure provides some nice abstraction capabilities on top, but building complex things from simpler component parts is what good software design is all about; suddenly classes holding tons of state and diving up and down method hierarchies feels like an unnecessary exercise.

## "Dependency Injection"
At Framed, our software was modularized using a lightweight version of dependency injection. I use those words carefully, because I think for a lot of people those words conjure mental images of factories and `@Inject` and Spring and all sorts of unpleasant things (unpleasant to me at least). We wrote thin clients for external services using Clojure [protocols](http://clojure.org/reference/protocols) (think Java interfaces), and passed them around as parameters. No magic required. In cases where we had a bunch of dependency instantiations, we'd put them in a map and pass it in as a single unit:

<pre class="prettyprint lang-clojure"><code>(ns storage.core)

(defprotocol Storage
  (put [this k v])</code></pre>

<pre class="prettyprint lang-clojure"><code>(deftype S3Client []
  storage.core/Storage
  (put [this k v]
    ; Call out to real S3 SDK...
  ))

(let [system
      {:storage (S3Client.)
       :redis (RedisClient.)
       :conn (datomic.api/connect my-uri)}]
  (do-work system foo bar))

(defn do-work [system foo bar]
  (let [storage-client (:storage system)]
    (storage.core/put storage-client "key" "value")))</code></pre>

Given a `Storage` protocol with a `put` function and an `S3Client` type that implements that protocol, the use of the `storage.core/put` function here dynamically dispatches to the proper implementation; the code has no idea if its going to hit the network, or if its just a mock version, or anything else.

(This is especially nice with [Datomic](http://www.datomic.com/). I highly recommend [The database as a value](http://www.infoq.com/presentations/Datomic-Database-Value). I digress.)

Testing code like this is trivial - we would write mock clients satisfying our protocols that operated purely on in-memory data or returned preset values, and pass them in; our code wouldn't know the difference. No more mocking out the network or just hoping it works on staging or production.

This approach isn't perfect, certainly. The overhead of writing a new protocol and wiring up real/mock clients for new services is surprisingly trivial and didn't really factor into our decision process. However when you're passing around dependency maps in Clojure, you have no guarantees about what it does or doesn't contain, so its up to you to carefully construct them and check things at runtime; that's kind of the rub with dynamic languages though. All in all this worked fantastically for us and was probably one of the most important technical decisions we made.

This section is slightly Clojure-specific, although similar things can be achieved with Java interfaces as mentioned, traits, or duck-typing depending on your language's toolbox choice. Design for swappability!

## Testing
Testing is a breeze when you program in a functional style, where a function operates solely on its inputs and produces a deterministic output*. Lightweight dependency injection as mentioned allowed us to control the state of the external world and interactions with it. Taken together, pretty much all you need to write tests is 'given this input, I get this output'. This equals that. Mocking, stubbing, doubles, fancy DSLs, and all the rest quickly become bloated, fragile, overbearing constructs. The built-in `clojure.test` ships with `is`, so you'd write 

 
<pre class="prettyprint lang-clojure"><code>(deftest test-add
  (is (= 5 (calc/add 2 3))))</code></pre>

This scales up remarkably well.

\* Clojure is not a pure language and we are not purists, but you can push this surprisingly far. We even did some fancy work with deterministically computing random number seeds and passing around `java.util.Random` instances so even our randomness was deterministic!

A huge red flag for me is if code needs to be run in some implicit context to be tested, because it is *incredibly* opaque and difficult to figure out exactly what that context needs to look like. By this, I mean everything from [needing to freeze the clock](https://github.com/travisjeffery/timecop) instead of just passing a timestamp as a parameter, mocking out every detail of network requests, or stubbing a dozen methods in advance before you can call the code under test.

We still had a few pretty hairy tests at Framed where we hadn't aggressively split apart certain parts of the code and it took a fair amount of legwork to set up the various states of our mock dependency world just so. Despite this, the fact that you _could_ test an entire production pipeline without ever interacting with the real outside world or reaching in to stub any functions remained very impressive to me.

## Technical debt

It's not to say that we didn't have *any* technical debt. In fact, it's been claimed that it's ["irresponsible for a startup not to have any technical debt"](http://firstround.com/review/shims-jigs-and-other-woodworking-concepts-to-conquer-technical-debt/). Especially in the early days of the company when things were uncertain and the future was hazy, a much more cavalier attitude was taken, and some things were done hastily or some suboptimal approaches taken that stuck with us for a while. Once things settled down a bit though, we took careful thought much more seriously and really took the time to get things right; _very_ little tech debt was introduced after those early days. I believe this led to massive overall time savings (as well as making the codebase a joy to work in!). All technical debt is born with the phrase "we'll just ship this hack/temporary solution/quick first version and revisit it later". Later never comes, of course. By the time you've rolled out your quick fix or hasty first attempt, there's a million other things to do and you can't waste time doing silly things like paying down technical debt. And that means you're stuck trying to build on top of a subpar foundation, stacking hacks and workarounds on each other and suddenly you're spending more time wrangling this mess than if you had just spent the few extra minutes to get it right at the start.

If things get bad enough, you might even end up with a Grand Rewrite, which will totally-solve-everything-this-time-I-promise (tm). Grand Rewrites are messy and error-prone beasts that can take weeks or months or longer to complete, way more time than that initial refinement phase that got skipped in the name of "ship it". Rewrites and migrations of large systems from old to new are black holes of complexity and error - avoid them like the plague. The "long time" in "eh, we won't need that for a long time" is often not so far off.

Most surreal of all is that in my experience this rinse-and-repeat cycle seems to be generally accepted as the cost of doing business. I feel like I've seen the same situation a dozen times over where everyone half-jokingly groans about the state of affairs, which is a looming quickly-cobbled together ball of mud and twine plagued by the occasional big migration of some component. It's possible to avoid a great deal of this by moving deliberately and thoughtfully; this is usually obvious in hindsight, and *tremendously* difficult to commit to at the outset. As a CTO or tech lead or what have you, it might feel contrary to every fiber in your being to say things like "no, take more time on this" or "we'll ship it when we get it right". It's also frustrating to hear this in review as an engineer itching to release their feature into the wild. Fight your instincts and move fast by moving slow.

There's a line of course. There are plenty of cases where you really don't need a fully-generic infinitely-scalable tower of abstraction up front. Sometimes its okay to ship now and defer things until later, or migrate an old system to something newer based on evolving requirements. It requires judgement, like everything else. All I'm saying is the need to ship that quick hack right-this-second-now is often less pressing than it seems.

## Wrapping up

These few points had massive influence on the simplicity of the Framed platform. I'd like to write more in the future about some more Clojure-specific things we did, but I think these ideas are broadly applicable to software in any language. It's sometimes surprising what you can achieve with a small number of well-designed building blocks, and my perspective has completely changed looking back on large object-oriented systems that rely heavily on stubbing and the like. I'd be fascinated to hear any opinions on the subject. Happy coding!