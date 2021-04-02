---
layout: post
title: "Suggesting URL aliases with CoffeeScript"
date: "2012-05-28"
permalink: /blog/post/suggesting-url-aliases-with-coffeescript
---

<p>
If you're familiar with any blogging platform or content management system, you've likely heard the term 'URL alias'. But what exactly does this refer to, and what problem does it solve? Let's start by looking at some common examples. If you've ever used Drupal, something like the path <code>node/369</code> might look familiar to you. Then consider some example post on this site, accessible through the path <code>andrewberls.com/blog/post/1234</code>. What's happening in both of these schemes is that we're accessing a piece of content based on its ID in a database. However, these URLs are not particularly revealing - the difference between <code>node/369</code> and <code>node/254</code> is a mystery as far as we're concerned. Instead, we'd like to define URLs with a human-readable structure that describes a page's content. This is what a URL alias is -  a readable path that allows us to access a resource. 
We'd like to create aliases for our content so that instead of using database IDs, we can using something more descriptive, perhaps using the content title to get something of the form <code>andrewberls.com/blog/post/my-awesome-post</code>. Implementing an aliasing scheme is beyond the scope of this article -  instead, this is a small feature I wrote for my blog which automatically suggests paths based on a post's title.
</p>

<break />

<h3>Suggesting a path</h3>
<p>
The algorithm here is relatively straightforward. Let's say that when we're writing a new post, we're given input fields for the title, body, and URL alias. We're given the option to fill in the alias manually if we'd like, but the task is sort of a pain so we'd like to cover it automatically. When the user enters a title, we have to slice it up and make it 'safe' for a URL, and fill the alias field with the URL. Let's get our setup out of the way:
</p>

<pre><code><span class="comment">&lt;!-- HTML --&gt;</span>
&lt;form&gt;
 &lt;input type=&quot;text&quot; id=&quot;title&quot; placeholder=&quot;Title&quot;&gt;
 &lt;textarea placeholder=&quot;Body&quot;&gt;&lt;/textarea&gt;
 &lt;input type=&quot;text&quot; id=&quot;url_alias&quot;&gt;
&lt;/form&gt;

<span class="comment"># CoffeeScript</span>
$ ->
  $("#title").bind "propertychange keyup input paste", ->
    url = $(@).val()
    $("#url_alias").val urlsafe(url)
</code></pre>

<p>
What are we accomplishing here? We're binding some anonymous function to a mess of events on the title field. When I first starting writing this, I just used the <code>change()</code> event on the title field, which was only called when the field lost focus (i.e., you enter a title and move on the body). This still worked, but we can do better than that. It would be cool if we filled in the URL alias as the user types the title. By binding our function to all these events, we're monitoring the title field in real time. But what does this anonymous function do? It starts by grabbing the value of the title field. The <code>$(@)</code> bit looks a bit strange though. In CoffeeScript, <code>@</code> is a shortcut for <code>this</code>. As a result, you can use <code>@property</code> as a shorthand for <code>this.property</code>, and <code>$(@)</code> is the same as <code>$(this)</code>. Once we have the title, it gets passed into some method called <code>urlsafe()</code> before filling in the alias field. This isn't a built-in method though - we're going to have to define it ourselves! 
</p>

<h3>The urlsafe() method</h3>
<p>We need to define a method called <code>urlsafe</code> that takes in a string and makes it safe for use as a URL. I'll start with another code dump, and then walk through what's going on.</p>
<pre><code>urlsafe = (url) ->
  url = $.trim url
  url = url.replace(/\s+/g, ' ')
  url = url.replace(/[\.,-\/#!$@%\^\*;&:\[\]{}=\-_`~()]/g, '')
  url.split(' ').join('-').toLowerCase()
</code></pre>

<p>
This method looks confusing at first, but we'll see that its nothing to be afraid of. We need to think about our goal - what exactly constitutes 'safe' for a URL? For starters, we're going to want to get rid of any whitespace such as spaces or tabs. We start by removing any whitespace at the beginning or end using <a href="http://api.jquery.com/jQuery.trim/">jQuery.trim()</a>. The first call to <a href="https://developer.mozilla.org/en/JavaScript/Reference/Global_Objects/String/replace">replace()</a> after <code>trim()</code> covers the edge case of repeated spaces inside the url, and compacts them down to a single space. This is unlikely to occur and not entirely necessary, but we might as well be on the safe side. The next call to <code>replace()</code> involves an ugly-looking regex that's simple in purpose - we're just removing any punctuation. The chained call to <code>split(' ').join('-')</code> replaces spaces with hyphens by first splitting the url into an array, separating tokens by spaces, and then just joining it back into a string with hyphens between tokens. Finally, we lowercase the entire thing and return the now-safe URL!
</p>

<h3>The result</h3>
<p>
Everything works as expected - the alias field updates in real time, and if you are compelled for reason to name title your post <code>&nbsp;&nbsp;this&nbsp;&nbsp;&nbsp;&nbsp;is@%!*#& the TITLE</code>, the suggestion comes out to the friendly <code>this-is-the-title</code>!
</p>
