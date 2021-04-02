---
layout: post
title: "A fresh coat of paint"
date: "2012-05-02"
permalink: /blog/post/fresh-coat-of-paint
---

<p>
I'm happy to unveil a much needed makeover for andrewberls.com! I've learned a huge amount since I first started this site what <a href="/blog/post/hello-world">seems like forever ago</a>, and I figured it was time to update the site to reflect that.
</p>

<p>
The process began with a much needed migration to Heroku's <a href="https://devcenter.heroku.com/articles/cedar">Cedar stack</a>. Cedar is the successor to the previous Bamboo runtime stack on which my old site was deployed, and the migration was actually surprisingly easy with the help of this <a href="https://devcenter.heroku.com/articles/cedar-migration">Dev Center article</a>. Theoretically, Cedar offers all sorts of cool things such as a process model and 'Twelve-factor methodology', but the most important thing for me is support for Rails' 3.1 asset pipeline. This means that I can take full advantage of Sass and CoffeeScript, and easily serve up concatenated/compressed assets in production.
</p>

<p>
My favorite part is that the new site is fully responsive, thanks to a <a href="http://andrewberls.github.com/Cheeky/">tiny grid framework</a> I wrote called Cheeky.css. Syntactically, Cheeky is very similar to other grid frameworks in that you contain content in <strong>rows</strong>, with 16 columns available for positioning. Once you've defined <strong>spans</strong> and <strong>offsets</strong>, Cheeky uses responsive media queries to automatically scale the page as the browser resizes. I'll write more about Cheeky later, but in the meantime you can read more about it <a href="http://andrewberls.github.com/Cheeky/">here</a>. Check it out - try resizing your browser on this page!
</p>

<break />

<h5>Tips and Tricks</h5>
<p>
I'm currently using Eric Martin's <a href="http://www.ericmmartin.com/projects/simplemodal/">Simple Modal</a> jQuery plugin for the modals on the front page. I've only played around with it a little bit, but it's been super easy to use and I recommend it.
</p>

<p>
I also made a new discovery - I'm convinced that <code>box-sizing: border-box;</code> is the best thing to ever happen to CSS. In the traditional box model, padding adds to the rendered width of an element. This can be confusing sometimes, as elements are wider than their declared widths. This is especially an issue if you want to add padding to an element of width: 100%. In the normal, or <code>content-box</code> box model, the element will actually break outside of the parent. Enter <code>box-sizing: border-box</code>! Elements with this property declared will render with their declared width, and padding/borders will cut inside the box. Now you can add padding to elements without breaking a grid layout, or have 100% width form inputs with padding but without ugly hacks!
</p>

<p>
Let's take a look at a very basic form example:
</p>

<pre><code><span class="comment">&lt;!-- The HTML --&gt;</span>
&lt;form class=&quot;contact-form&quot;&gt;
  &lt;input type=&quot;text&quot; placeholder=&quot;Name&quot;&gt;
  &lt;textarea placeholder=&quot;Enter your message&quot;&gt;&lt;/textarea&gt;
&lt;/form&gt;

<span class="comment">/* The CSS */</span>
.contact-form { padding: 20px; }

input[type="text"], textarea {
  /* The magic bit! */
  -webkit-box-sizing: border-box;
     -moz-box-sizing: border-box;
          box-sizing: border-box;

  margin-top: 15px; /* Space things out a bit */
  padding: 5px;
  width: 100%;
} 
</code></pre>

<p>
And now you'll have beautiful inputs with padding that fill the content area! Hooray! We're using the <code>[type="text"]</code> selector here in order to exclude <code>[type="submit"]</code>. Note that this is also using the <a href="http://davidwalsh.name/html5-placeholder">placeholder</a> attribute, a neat HTML5 feature.
</p>

<p>
To wrap things up, border-box is a property that can make the box-model a lot more intuitive. However, I'm still wary of using a rule like <code>* { box-sizing: border-box; }</code> as <a href="http://paulirish.com/2012/box-sizing-border-box-ftw/">described by Paul Irish</a>, and one has to keep in mind that browser prefixes are still necessary, so this should be reserved for projects supporting IE8 and up.
</p>



<h5>What's next</h5>
<p>
Now that things are nice and pretty on the front end, I plan to dive in and rewrite the blog engine that powers this site. I wrote it when I was first learning Rails, and there's much to be done in the way of refactoring and cleaning up my first attempt. I'll likely be taking a lot of inspiration from Nate Wienert's <a href="http://natewienert.com/codename-obtvse">Obtvse</a> project. As always, all the code is open source over at <a href="https://github.com/andrewberls/andrewberls">GitHub</a>. I'd love to hear any ideas or criticisms!
</p>

