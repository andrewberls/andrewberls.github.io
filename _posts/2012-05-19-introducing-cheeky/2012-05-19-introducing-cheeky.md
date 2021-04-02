---
layout: post
title: "Introducing Cheeky: The responsive grid with Sass"
date: "2012-05-19"
permalink: /blog/post/introducing-cheeky
---

<p>
<strong>Mobile development doesn't have to be hard.</strong> That's why I created my latest project - Cheeky.css, a grid system that makes building responsive sites a breeze. It's more important than ever to create mobile-friendly sites, but defining a separate version or stylesheet just for handhelds is a pain. With Cheeky, you only have to style your page once, and your content will automatically scale to fit the browser width.
</p>


<h3>Using Cheeky</h3>
<p>
If you've used any grid before, the syntax will be very familiar. Cheeky is based off an 1100px grid with 16 columns contained in rows. 16 is an arbitrary number, but combined with familiar column offsets, you have the freedom to structure your content exactly how you like. To get started, everything is unsurprisingly contained in a <code>&lt;div class=&quot;container&quot;&gt;</code>. From there, you lay out your columns inside a <code>&lt;div class=&quot;row&quot;&gt;</code>, 16 to a row. Columns are defined using the syntax <code>&lt;div class=&quot;span-#&quot;&gt;</code>, where # is any number between 1 and 16. Offsets are almost identical - just toss on a <code>class=&quot;offset-#&quot;</code>, except that you can only offset up to 15. Let's take a look at the bare minimum example:
</p>

<break />

<pre><code>&lt;div class=&quot;container&quot;&gt;
  &lt;div class=&quot;row&quot;&gt;
    &lt;div class=&quot;span-8&quot;&gt;This is 8 columns&lt;/div&gt;
    &lt;div class=&quot;span-4 offset-4&quot;&gt;This is 4 columns, offset by 4!&lt;/div&gt;
  &lt;/div&gt;
&lt;/div&gt;
</code></pre>

<p>
That's all there is to it! Once you get used to working with spans and offsets, structuring your content is easy and Cheeky does all the work for you. Nested columns are totally possible as well - just stick in another row and structure nested content the same way. You can find more usage information and downloads <a href="http://andrewberls.github.com/Cheeky/">here</a>!
</p>


<h3>Details</h3>
<p>
As mentioned before, Cheeky's base is an 1100px grid for desktop screens. Once the browser is resized, CSS media queries are used to progressively scale the grid down to 790px and 630px (In the code, these two states are listed as 'small' and 'tiny' grids). Past that point (portrait phones and below), Cheeky reverts to displaying columns stacked on top of each other. Cheeky is built with Sass (hence the tagline. Don't be misled; it's actually very friendly), so the entire grid system can be customized and tweaked just by changing a few variables.
</p>


<h3>Tips and Tricks</h3>
<p>
I learned a great deal about grid-based design and CSS media queries writing Cheeky, and I wanted to pass on a neat Sass trick I just discovered. For example, I use a mixin called <code>width</code> when defining spans, which takes in an integer and uses it to calculate the correct width. I had written a bunch of code that looked like this:
</p>

<pre><code>.span-1  { @include width(1); }
.span-2  { @include width(2); }
.span-3  { @include width(3); }
.span-4  { @include width(4); }
.span-5  { @include width(5); }
<span class="comment">/*...on and on, up to 16 */</span>
</code></pre>

<p>
This went on and on for spans and offsets in all of the various media queries, creating a ton of repetitive code. Then I discovered that it's a perfect candidate for refactoring using a Sass loop, which reduces the example above to something like this: 
</p>

<pre><code>@for $i from 1 to 17 {
  .span-#{$i} { @include width($i); }
}
</code></pre>

<p>
That by itself generates the code for 16 classes (it doesn't include the endpoint) including the proper mixin! Turns out that <code>for</code> is only one of the Sass control directives; <code>if</code>, <code>each</code>, and <code>while</code> are also included. I don't have too much experience with them yet, but you can read more about them <a href="http://thesassway.com/intermediate/if-for-each-while">here</a>.
</p>


<h3>Why?</h3>
<p>
With a large number of grid systems already available, one might ask why bother writing another one. Cheeky was created entirely as a learning exercise to practice responsive design techniques, and derives much inspiration from the excellent Twitter <a href="http://github.com/twitter/bootstrap">Bootstrap</a> project. I'm using Cheeky in production on this site (resize this window!) and several other projects, and it's held up to the task. If you'd like to read more about Cheeky, I've set up a more extensive info page <a href="http://andrewberls.github.com/Cheeky/">here</a>, and the project is hosted on GitHub <a href="http://github.com/andrewberls/Cheeky">here</a>. Finally, if you use Cheeky in any of your projects be sure to let me know on <a href="http://twitter.com/aberls">Twitter</a>!
</p>

<p><em>Update 5/22: As pointed out in the comments, some of the media queries were causing a horizontal scrollbar at certain dimensions. The framework has since been updated to fix this issue.</em></p>
