---
layout: post
title: "JavaScript tricks: Enforcing function arity"
date: "2013-05-26"
permalink: /blog/post/javascript-tricks-enforcing-function-arity
---

I was browsing through the source code of the [JavaScript driver for RethinkDB](https://github.com/rethinkdb/rethinkdb/tree/next/drivers/javascript/src) the other day, and I came across a clever pattern that was too good not to write about. [RethinkDB](http://www.rethinkdb.com/) is an open-source JSON document store that I've heard nothing but good things about, but unfortunately I haven't spent too much time with it myself. Instead, this post is about a JavaScript pattern in their driver code, which is actually implemented in CoffeeScript. Let's take a look.

<a href="/images/posts/ar_use.png"><img src="/images/posts/ar_use.png" alt="ar function definition" /></a>
<p class="img-caption">Click for larger image</p>

<break />

I noticed many of their driver methods are defined with this weird `ar` syntax, as seen in the first line. I consider myself fairly familiar with CoffeeScript but this one stumped me for a bit - is it a keyword? Some weird piece of CoffeeScript method definition syntax that I missed out on? Some snooping revealed that `ar` is actually an ordinary method defined in [drivers/javascript/src/base.coffee](https://github.com/rethinkdb/rethinkdb/blob/next/drivers/javascript/src/base.coffee#L11). Here's the definition:

<a href="/images/posts/ar_def.png"><img src="/images/posts/ar_def.png" alt="ar function definition" /></a>
<p class="img-caption">Click for larger image</p>

From the description: "function wrapper that enforces that the function is called with the correct number of arguments". Neat! But how does it work? Let's deconstruct that first line. 

```
ar = (fun) -> (args...) ->
```

This looks pretty unusual at first glance, but if we recall how CoffeeScript works, functions will always return their final value. Hence, `ar` is a function that takes a function as a parameter, and returns a function in turn. This can be a little confusing in CoffeeScript syntax, so let's do things explicitly in Javascript.

<pre class="prettyprint lang-js"><code>function ar(fun) {
  return function(args) {
    // something here ...
  }
}</code></pre>

As it turns out, the `args` parameter in the inner function isn't quite right because of how CoffeeScript handles splats. We'll cover that later - let's look at the inside of the `ar` function first.

What `ar` does is check the number of parameters that you're calling a function with and compare it with the number of parameters that it expects (called the function [arity](https://en.wikipedia.org/wiki/Arity) - oooh!), and raises an error if they don't match. But how do you know how many arguments a function expects? This is the coolest part of this code. As it turns out, you can do this:

<pre class="prettyprint lang-js"><code>function test(a, b) {}
test.length // => 2
</code></pre>

We saw earlier that `ar` takes a function and wraps it with an argument checker. If the number of arguments match up, it just passes arguments through to its wrapped function. Let's look at one final explicit example:

<pre class="prettyprint lang-js"><code>function greet(a,b) {
  console.log("Hello " + a + " " + b + "!");
}

function argCheck(func) {
  return function() {
    if (arguments.length !== func.length) {
      throw new Error("Argument mismatch! Expected: " + func.length +
        ", received: " + arguments.length);
    }

    return func.apply(this, arguments);
  }
}

var someObject = {
  sayHello: argCheck(greet)
};

someObject.sayHello("gentle", "readers");
// => Hello gentle readers!
</code></pre>

In this example `someObject` is an object with a single method `sayHello` that takes exactly two arguments and prints a message to the console.  We're using the `arguments` object instead of explicitly passing parameters into the inner function, as this is how CoffeeScripts `...` splat works internally and allows us to accept variable length arguments. 

Now let's see what happens if we call `sayHello` with an incorrect number of arguments:

<pre class="prettyprint lang-js"><code>myObject.sayHello("gentle");
// => Uncaught Error: Argument mismatch! Expected: 2, received: 1 
</code></pre>

Just as expected, we'll get an error if we don't pass in exactly 2 arguments, as expected in the definition of `greet` (which `sayHello` is wrapping with `argCheck`)

Hopefully it's a little clearer what `ar` does at this point. It's a clever way of enforcing function arity, and keeps code very [DRY](http://en.wikipedia.org/wiki/Don't_repeat_yourself). Although examples of JavaScript may be more illustrative, the CoffeeScript code in the driver is much less obtrusive and more elegant.
