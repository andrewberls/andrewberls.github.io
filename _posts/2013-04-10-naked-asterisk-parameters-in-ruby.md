---
layout: post
title: "Naked asterisk parameters in Ruby"
date: "2013-04-10"
permalink: /blog/post/naked-asterisk-parameters-in-ruby
---

I came across a Ruby idiom I hadn't seen before browing through some of the [Rails source](https://github.com/rails/rails/blob/master/activerecord/lib/active_record/persistence.rb#L102) the other day. Specifically, the method in question looked like this:

<pre class='prettyprint lang-ruby'>
<code>def save(*)
    create_or_update
rescue ActiveRecord::RecordInvalid
    false
 end</code></pre>

I've never seen the asterisk as a parameter by itself, and to explain it a brief review of the [splat operator](#) is in order. The splat operator allows methods in Ruby to accept a variable number of arguments. For example:

<pre class='prettyprint lang-ruby'>
<code>def say_hello(*people)
  people.each { |person| puts "Hello #{person}!" }
end
 
say_hello("Alice", "Bob", "John")
# "Hello Alice!"
# "Hello Bob!"
# "Hello John!"</code></pre>

<break />

The splat can be used with named arguments as well - it will just gather up all of the unnamed arguments into an array. It can also be used for a couple other really neat tricks, which are explained in detail [in this post](http://endofline.wordpress.com/2011/01/21/the-strange-ruby-splat/). However,when using the splat, it's much more common to see a definition like

<pre class='prettyprint lang-ruby'>
<code>def some_method(*args)
  # do something with args
end</code></pre>

So what's the deal with the unnamed asterisk? Well, this is still the same splat operator at work, the difference being that it does not name the argument array. This may not seem useful, but it actually comes into play when you're not referring to the arguments inside the function, and instead calling `super`. To experiment, I put together this quick snippet:

<pre class='prettyprint lang-ruby'>
<code>class Bar
  def save(*args)
    puts "Args passed to superclass Bar#save: #{args.inspect}"
  end
end
 
class Foo < Bar
  def save(*)
    puts "Entered Foo#save"
    super
  end
end
 
Foo.new.save("one", "two")</code></pre>

which will print out the following:

    Entered Foo#save
    Args passed to superclass Bar#save: ["one", "two"]

The globbed arguments are automatically passed up to the superclass method. Some might be of the opinion that it's better to be explicit and define methods with `def method(*args)`, but in any case it's a neat trick worth knowing!