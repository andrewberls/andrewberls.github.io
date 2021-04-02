---
layout: post
title: "Ruby tips: Procs vs lambdas"
date: "2013-05-14"
permalink: /blog/post/ruby-tips-procs-vs-lambdas
---

I recently ran into a Ruby subtlety that I had heard of but never encountered before, regarding the use of the `return` keyword within blocks. Let's jump right into an example. Let's say you have an array of numbers and you want to [select](http://ruby-doc.org/core-2.0/Enumerable.html#method-i-select) all the even ones. What will the following output?

<pre class='prettyprint lang-ruby'><code>def evensWithReturn(items)
  items.select do |item|
    if item.even?
      return true
    else
      return false
    end
  end
end

> evensWithReturn([1,2,3,4,5,6])
# => false
</code></pre>

<break />

Wait, what? We're expecting to see `[2, 4, 6]`, not `false`. What's going on here? Let's try it again, this time removing the `return` keyword from the `select` block.

<pre class='prettyprint lang-ruby'><code>def evensWithoutReturn(items)
  items.select do |item|
    if item.even?
      true
    else
      false
    end
  end
end

> evensWithoutReturn([1,2,3,4,5,6])
# => [2, 4, 6]
</code></pre>


Much better. Although why was the `return` keyword messing things up? It turns out that this is a result of the subtle differences between Procs and lambdas in Ruby, both of which represent chunks of reusable code. For the most part, the two are identical, although one of the differences is that <span class="bold">a Proc returns from its enclosing block/method</span>, while <span class="bold">a lamba simply returns from the lambda</span>. Let's see some more examples. First, using Procs:

<pre class='prettyprint lang-ruby'><code>def procWithReturn(items)
  myProc = Proc.new do |item|
    if item.even?
      return true
    else
      return false
    end
  end

  items.select(&myProc)
end

> procWithReturn([1,2,3,4,5,6])
# => false
</code></pre>

This is the same result as we saw with the block. Still not the result we wanted, but at least we know why. As soon as the proc gets the first item (1), it will call `.even?` on it and return false, and consequently return false from the entire `procWithReturn` method. However, it we try the same example using a lambda instead of a proc:

<pre class='prettyprint lang-ruby'><code>def lambdaWithReturn(items)
  myLambda = lambda do |item|
    if item.even?
      return true
    else
      return false
    end
  end

  items.select(&myLambda)
end

> lambdaWithReturn([1,2,3,4,5,6])
# => [2, 4, 6]
</code></pre>

Success! Here, the lambda returns values to the `select` block, but doesn't exit the enclosing method. For completeness, the other difference between Procs and lambdas is that lambdas enforce <em>arity</em>, meaning it will throw an `ArgumentError` if you don't pass the number of arguments it specifies.

For the record, this example is fairly contrived for illustrative purposes - before you jump down my throat, I know that you can do

<pre class='prettyprint lang-ruby'><code>[1,2,3,4,5,6].select(&:even?)
</code></pre>

I hope this clarifies some of the subtleties between procs and lambdas. For interested readers, I recommend [this excellent blog post](http://www.robertsosinski.com/2008/12/21/understanding-ruby-blocks-procs-and-lambdas/) which goes into further detail.