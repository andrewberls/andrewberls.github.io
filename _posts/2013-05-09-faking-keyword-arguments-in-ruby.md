---
layout: post
title: "Faking keyword arguments in Ruby"
date: "2013-05-09"
permalink: /blog/post/faking-keyword-arguments-in-ruby
---

With the official release of [Ruby 2.0](http://www.ruby-lang.org/en/news/2013/02/24/ruby-2-0-0-p0-is-released/), there have been a myriad of posts about all of its shiny new features. Of those, my personal favorite is the addition of [keyword arguments](http://globaldev.co.uk/2013/03/ruby-2-0-0-in-detail/). This is essentially built-in support for what's been traditionally accomplished in Ruby using options hashes.  For example, it's not uncommon to see method calls like <code class="prettyprint lang-ruby">foo(bar: "one", baz: "two")</code>. However, under the hood this might be implemented as:

<pre class="prettyprint lang-ruby"><code>def foo(options={})
  bar = options.delete(:bar)
  baz = options.delete(:baz)
  puts "#{bar} #{baz}"
end

foo(bar: "one", baz: "two") # => "one two"
</code></pre>

<break />




In fact [ActiveSupport in Rails 3.2.13](https://github.com/rails/rails/blob/3-2-13/activesupport/lib/active_support/core_ext/array/extract_options.rb) defines a method called `extract_options!` that deals with exactly this. However, if we're stuck on Ruby < 2.0, defining every method to use an options hash is often more work than necessary, especially when a method only takes one or two parameters. We can get around this with a clever trick. If we change our method slightly from the previous example to work without an options hash:

<pre class="prettyprint lang-ruby"><code>def foo(bar, baz)
  puts "#{bar} #{baz}"
end</code></pre>

We can then fake keyword arguments by calling it like this: 

<pre class="prettyprint lang-ruby"><code>foo(bar = "one", baz = "two")
# => "one two"
</code></pre>

This trick exploits the fact that the returned value from the assignments in the call to <code class="prettyprint lang-ruby">foo()</code> will be the value that's assigned, so at the end of the day <code class="prettyprint lang-ruby">"one"</code> and <code class="prettyprint lang-ruby">"two"</code> will still be passed in! This saves us having to work with options hashes and still looks pretty, although it's important to note that unlike hashes and Ruby 2 keyword arguments, order matters here. If you change up the order you pass in `baz` and `bar`, things won't work as expected. 

Personally, I can't wait to switch my projects to use Ruby 2.0 and use keywords arguments for real, but in the mean time this is a neat trick I hadn't seen before to emulate them!