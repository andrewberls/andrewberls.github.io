---
layout: post
title: "Partial function application for humans"
date: "2014-11-08"
permalink: /blog/post/partial-function-application-for-humans
---

If you've ever spent any time reading about functional programming, you may have heard about "partial function application" or "currying". For the longest time these were just big scary words to me, but it turns out they're relatively simple concepts.  I've read plenty of posts describing what they *are* , but it wasn't until recently that I really came to grasp *why* one would ever need them. In this post, I'll briefly go over how partial application/currying work (and the subtle difference between them!), but I mostly want to talk about why you should care and when they can be used.

<break />

## The what

Recall that *arity* refers to the number of arguments that a function accepts - `add(a, b)` has an arity of 2. From [Wikipedia](https://en.wikipedia.org/wiki/Partial_application), partial application is the process of fixing some number of arguments to a function, thus producing a function of smaller arity. Let's illustrate this in Ruby with procs and the [Proc#curry](http://ruby-doc.org/core-2.1.4/Proc.html#method-i-curry) method. As I mentioned, there is a subtle difference between partial application and currying, which will be explained later.


Here's a simple proc in Ruby which adds two numbers:

<pre class="prettyprint lang-ruby"><code>add = proc { |x, y| x + y }
add.call(2, 3) # => 5</code></pre>

This is a proc taking two arguments. However, using the `Proc#curry` method, we can call it with fewer than two arguments, and it will return a proc that takes the rest of the arguments. Here's what that looks like:

<pre class="prettyprint lang-ruby"><code>add = proc { |x, y| x + y }
add_one = add.curry.call(1) # => #&lt;Proc:0x00...&gt;
add_one.call(3) # => 4
add_one.call(9) # => 10</code></pre>

By supplying only one of the arguments to the curried `add` proc, we get *another* proc that in turn accepts the last argument and adds the two numbers for us. Since `add` expects two arguments, and we only supplied one, this is called *partially applying* the `add` function (in mathematics, calling a function is referred to as  "applying the function to its arguments").  This works for any number of arguments - if we had a function taking 3 arguments, we could supply 2 and get back a function expecting 1, and so on.

Now, most of the posts I've read about partial application stop there. At this point, we get the gist of what partial application is, but we haven't accomplished anything since nobody uses procs like this and adding two numbers is not a very revolutionary takeaway.

Let's nail down the difference between partial application and currying before we dive into why any of this matters.

### Currying

For the longest time, I thought partial application and [currying](https://en.wikipedia.org/wiki/Currying) were the same thing - the terms seems to show up interchangeably in functional programming literature, and the difference is not huge. We already saw that partial application refers to supplying some number of arguments to a function, and getting back a new function that takes the rest of the arguments and returns the final result. *Currying* refers to the process of taking some function that accepts multiple arguments, and turning it into a sequence of functions, each accepting a single argument. In notation, a function `f` might look like

<pre><code>f: (x, y, z) -> n</code></pre>

`f` accepts 3 arguments (`x`, `y`, and `z`), and returns some value `n`. Currying `f` would yield

<pre><code>curry(f): x -> (y -> (z -> n))</code></pre>

The curried version turns `f` into a sequence of *3 separate functions*, each taking 1 argument. While we might call the uncurried version as `f(1, 2, 3)`, the curried version would be called as `f(1)(2)(3)`.

The difference is this: recall that for partial application, we could supply `f` with one argument, and get back another function that would accept the last 2 arguments and return the result immediately when called with those 2 arguments. Currying refers to the process of breaking `f` down into a chain of 3 functions, each taking only a single successive argument, before returning the final answer. Like I said, the difference is subtle. In Ruby, the `Proc#curry` method does both of these things. [Wikipedia](https://en.wikipedia.org/wiki/Currying#Contrast_with_partial_function_application) has a good contrast between the two, and Reginald Braithwaite has a [great post](http://raganwald.com/2013/03/07/currying-and-partial-application.html) which explains the differences in more detail.


## The why

I hope I haven't lost you at this point - as I said, the purpose of this post is to explain why you might care about any of this. The examples we've seen so far have not been terribly compelling. Here's my opinion on the subject: if you're programming in a language like Ruby or Javascript, these concepts are of limited use to you. Most of the examples I've read have been in these languages, and so it took me a long time to really get it. In a more functional language, the story is different.

There are many benefits to adopting a functional style in your code (in any language) that I won't enumerate here, but in a language that's not functional-first, these things will seem pretty forced. That's because these languages have different approaches and conventions for managing state and computation, and trying to curry a bunch of procs in Ruby for example goes against the more "normal" way of doing the same task.

One of the primary focuses of functional programming is state - it tries to avoid maintaining global state and mutating any state as much as possible, and instead operates by composing 'pure' functions, which operate *only* on their inputs. These functions are then like easy-to-understand building blocks that can be used to construct a more complex program. Yada yada.

Back to partial application. Here's the realization that got me on board with everything: __in functional programming, state is often maintained in function parameters.__ This is why recursion shows up more in FP - it represents an alternative way to 'iterate' and build up some state stored in an accumulator parameter. This also makes things like partial application make a whole lot more sense.

Let's see an example! At [Framed](http://framed.io), we write a lot of [Clojure](http://clojure.org/), so the examples will be in Clojure but you can probably follow along even if you're not familiar with the language.

Framed computes a lot of things related to retention. For example, a customer might want to know, how many users visited my site, and then visited again 7 days later? We say that these users retained over a 7 day period. Let's write some functions to summarize this information.

We can write a function to check if any given user retained like this:

<pre class="prettyprint lang-clojure"><code>(defn user-retained? [retained-user-ids user-id]
  (contains? retained-user-ids user-id))</code></pre>

All this does is check if a given user id exists in some set of retained user ids. Here, the retained user id set is a parameter, even though it's really more like fixed state. See where this is going?

We can partially apply this function with some set of retained user ids, and get back a new function that will take a user id and tell us if they retained. If we make up sample data for retained users and some set of users in question, we can run this function over the whole sequence like so:

<pre class="prettyprint lang-clojure"><code>(def retained-ids #{3 4 5}) ; #{} constructs a Set
(def user-ids [1 2 3 5 6])

(map (partial user-retained? retained-ids) user-ids)
; => (false false true true false)
</code></pre>

In Clojure, `partial` takes a function and some arguments, and works just like `Proc#curry` in Ruby. Here we partially apply it with the `retained-ids`, and get back a new function that will take a single user id, passed to it as we `map` over the sequence (the signature of map is `(map fn collection)`).

For fun, we can call the built-in `frequencies` function to count up the results:

<pre class="prettyprint lang-clojure"><code>(frequencies (map (partial user-retained? retained-user-ids) user-ids))
; => {false 3, true 2}</code></pre>

If any hardcore Clojure-ists are reading this, there are even more elegant ways of writing that, but that's not really the point of this post.

For another example, if we wanted to fetch a bunch of user data given their ids out of [Redis](http://redis.io/), we might write

<pre class="prettyprint lang-clojure"><code>(defn fetch-user [conn user-id]
  (redis/get conn user-id))

(defn fetch-users [conn user-ids]
  (map (partial fetch-user conn) user-ids))</code></pre>

In Ruby we might have had a global `$redis` object hanging around, whereas here we pass a connection around as 'state', and thus it's handy to partially apply things with it.

The list of examples goes on and on. Anyway, so far we've seen a very long-winded explanation of how partial application is useful in functional programming to fix some 'state' parameters in a function, and get back a new function that can operate on variable data. That's not the only way in which its useful, but its the pattern I've seen most often so far, and which really made things click for me.

I hope I've succeeded in explaining when and why you might care about things partial application and currying. If you are a developer in a non-predominantly-functional-language, fear not; these may be of use of you yet, and I'd be very interested to see examples in practice. And for what it's worth, [Clojure](http://clojure.org/) is a very fun language worth checking out. Happy currying!