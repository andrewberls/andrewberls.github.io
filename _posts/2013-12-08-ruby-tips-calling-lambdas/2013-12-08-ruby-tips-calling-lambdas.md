---
layout: post
title: "Ruby tips: Calling lambdas"
date: "2013-12-08"
permalink: /blog/post/ruby-tips-calling-lambdas
---

Lambdas are a powerful way to write anonymous functions (or closures) in Ruby. I wrote about the subtle differences between lambdas and procs [here](/blog/post/ruby-tips-procs-vs-lambdas), and this post is about something a little different. I recently discovered a number of different ways to invoke lambdas that I had never seen before, and ended up learning something new about Ruby itself.

Let's look at a basic example. Here is a lambda that doesn't do much of anything:

<pre class='prettyprint lang-ruby'><code>test = lambda do |*args|
  p args
end
</code></pre>

So we have a lambda `test` that prints its arguments when invoked. The most basic example of this would be something like:

<break />

<pre class='prettyprint lang-ruby'><code>test.call(:one, :two) # => prints [:one, :two]
</code></pre>

Nothing surprising there - `#call` is the standard method for invoking a lambda. We can also invoke lambdas with `[]`:

<pre class='prettyprint lang-ruby'><code>test[:three, :four] # => prints [:three, :four]
</code></pre>


All was well until I came upon [this](http://eyeofthesquid.com/blog/2013/12/05/juxtaposition-in-ruby/) blog post. Looking at line 12 of the first Ruby example, we see this:

<pre class='prettyprint lang-ruby'><code>juxt.(:+, max, min).(2, 3, 5, 1, 6, 4)
</code></pre>

Ignoring the functionality of the `juxt` lambda for now (although the post is highly recommended reading), the syntax caused me to do a double take. `juxt.(:+, max, min)` is unlike any Ruby I've seen before. A little experimentation reveals that `.()` is just syntactic sugar for the `#call` method! This means that we can call 

<pre class='prettyprint lang-ruby'><code>test.(:five, :six) # => prints [:five, :six]
</code></pre>


How weird is that? As it turns out, it works for things besides lambdas as well. Take a look:

<pre class='prettyprint lang-ruby'><code>class Example
  def call(*args)
    p args
  end
end

Example.new.(:hello, :world) # => prints [:hello, :world]
</code></pre>


For some final weirdness, [this](http://stackoverflow.com/questions/19108550/how-does-rubys-operator-work) StackOverflow answer notes that the following is also valid:

<pre class='prettyprint lang-ruby'><code>test::(:seven, :eight) # => prints [:seven, :eight]
</code></pre>

We've seen a number of different ways to equivalently invoke the `#call` method on a lambda. I make no assertion about whether or not you should use these in practice, although it's fascinating to learn about the various intricacies of Ruby. All of these, and more goodies,  are tucked away in the Proc documentation [here](http://www.ruby-doc.org/core-2.0.0/Proc.html#method-i-call).


### Bonus

If you poke around the Proc documentation linked above, you'll notice that using `===` between a proc and an object will invoke the proc with the object as a parameter. That allows you to do this:

<pre class='prettyprint lang-ruby'><code>test === :nine # prints [:nine]
</code></pre>

While that technically works, I certainly wouldn't use that in practice. The real reason for this is allowing you to use procs in the `when` clause of a case statement, which delegates to `===`. Here's a better example:

<pre class='prettyprint lang-ruby'><code>is_even_int = lambda do |num|
  num.is_a?(Integer) && num.even?
end

x = 4

case x
when is_even_int
  puts 'x is an even integer'
else
  puts 'x is not an even integer'
end
</code></pre>

This concludes our tour of the myriad ways to invoke lambdas/procs. I'd be interested to hear any examples of ways people are using these in the wild. Happy `#call`ing!
