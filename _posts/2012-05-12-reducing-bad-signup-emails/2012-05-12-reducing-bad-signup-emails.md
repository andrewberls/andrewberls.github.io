---
layout: post
title: "Reducing bad signup emails with mailcheck.js"
date: "2012-05-12"
permalink: /blog/post/reducing-bad-signup-emails
---

<p>
I recently stumbled across <a href="https://github.com/kicksend/mailcheck">mailcheck.js</a>, a little jQuery plugin from <a href="http://kicksend.com/">Kicksend</a> that suggests domains based on common typos in email forms. For example, 'user@gnail.co' will generate a suggestion for 'user@gmail.com'. It's perfect for preventing errors in user signups, and the authors claim its reduced their email bounces by 50%. After playing around with it, I've decided to bundle it into production for most, if not all of my projects, and this is just a brief demo of what can be done with it.
</p>

<h3>Getting Set Up</h3>
<p>
Our goal is to create a simple display that shows email suggestions to the user and offers a way to automatically fill in the field with the suggestion.

The plugin is used in two steps - first, it has to be attached to a text field, and then you have to actually do something with its suggestions. Let's get the basic JavaScript structure set up (make sure you include the actual plugin on your page as well!)
</p>

<pre><code><span class="comment">&lt;!-- The HTML --&gt;</span>
&lt;input id=&quot;email&quot; type=&quot;text&quot; placeholder=&quot;Email&quot;&gt;
&lt;div id=&quot;hint&quot;&gt;&lt;/div&gt;


<span class="comment">// The JavaScript</span>
var $email = $('#email'); <span class="comment">// Cache jQuery objects into variables</span>
var $hint = $("#hint");

$email.on('blur', function() {
  $(this).mailcheck({
    suggested: function(element, suggestion) {

    }
  });
});
</code></pre>

<p>
The setup is only a few lines, although it's important to understand what's going on. Our HTML is fairly straightforward - we start with a generic input that we'll be using the plugin on. The <code>&lt;div id=&quot;hint&quot;&gt;</code> is an initially hidden div that will contain our suggestions to the user. In our JavaScript, we're first sticking jQuery objects into variables for convenience. Then we're attaching the <code>mailcheck()</code> function to our email field on its blur event, which is called whenever the element loses focus (i.e., the user moves on to the next field). Mailcheck takes two callbacks, <code>suggested()</code> and <code>empty()</code>. <code>suggested()</code> is called whenever a suggestion is available for the field, and <code>empty()</code> whenever the field is left blank. We'll only be using <code>suggested()</code> in this demo, although depending on how you use the plugin it's generally a good idea to use both.
</p>

<break />

<h3>Using the suggestion object</h3>
<p>
Mailcheck automatically generates our field suggestions for us, but how do we use them? Note that the <code>suggested()</code> method from before is passed in two parameters - <code>element</code>, which is the field that we're checking, and something called <code>suggestion</code>. If you enter user@gnail.co and use your favorite tool to inspect it, you'll notice it's an object with three fields:
</p>

<img src="/images/posts/mailcheck_object.png" alt="The mailcheck suggestion object" />

<p>
We have access to the address (what comes before the @), the suggested domain, and the entire suggested text. We can use these fields to populate and show a hint to the user using the previously mentioned <code>&lt;div id=&quot;hint&quot;&gt;</code>. Here's how you could go about filling the hint element:
</p>

<pre><code>var $email = $('#email');
var $hint = $("#hint");

$email.on('blur',function() {
  $hint.css('display', 'none'); <span class="comment">// Hide the hint</span>
  $(this).mailcheck({
    suggested: function(element, suggestion) {
      if(!$hint.html()) {
        <span class="comment">// First error - fill in/show entire hint element</span>
        var suggestion = &quot;Yikes! Did you mean &lt;span class='suggestion'&gt;&quot; +
                          &quot;&lt;span class='address'&gt;&quot; + suggestion.address + &quot;&lt;/span&gt;&quot;
                          + &quot;@&lt;a href='#' class='domain'&gt;&quot; + suggestion.domain + 
                          &quot;&lt;/a&gt;&lt;/span&gt;?&quot;;
                          
        $hint.html(suggestion).fadeIn(150);
      } else {
        <span class="comment">// Subsequent errors</span>
        $(".address").html(suggestion.address);
        $(".domain").html(suggestion.domain);
      }
    }
  });
});
</code></pre>

<p>
Before we do anything with the suggestion, we're making sure to hide the hint element. If there's a suggestion, we'll be repopulating it with the suggested domain, but this handles the case that a valid email is entered after we've already suggested a hint. At the beginning of the <code>suggested()</code> callback, we're checking to see if the hint element is empty (<code>if (!$hint.html()) {...}</code>). If so, then we can assume it's the user's first error. The suggestion variable is a rather nasty string of HTML that we'll be inserting into the hint element. There are other ways of doing this such as template systems or DOM libraries, but this is a quick and dirty solution. On the first error, we first fill in the hint with our suggestion (<code>$hint.html(suggestion)</code>) and fade it in by chaining it to jQuery's <a href="http://api.jquery.com/fadeIn/">fadeIn</a> method. On subsequent errors, instead of fading in the hint, all we have to do is modify its contents to reflect the new contents of the field. At this point, we should have a working hint!
</p>

<img src="/images/posts/mailcheck_suggestion.png" alt="Email suggestion to the user" />

<p>
At this point, we're missing the last piece of the puzzle - I've styled the domain to look like an inviting link, but clicking it doesn't actually do anything. It should be a link automatically fill in the suggestion for you - let's implement it!
</p>

<pre><code><span class="label">// After our other code:</span>
$hint.on('click', '.domain', function() {
  <span class="comment">// On click, fill in the field with the suggestion and remove the hint</span>
  $email.val($(".suggestion").text());
  $hint.fadeOut(200, function() {
    $(this).empty();
  });
  return false;
});
</code></pre>

<p>
The code is much shorter for this bit - we're using jQuery's <a href="http://api.jquery.com/on/">on()</a> method to attach a click handler to the <code>.domain</code> link. All we're doing is filling in the field with the suggestion text, and fading out the hint, emptying it when it's complete. And that's it! We have a fully functional hint element that displays suggestions to the user and provides a link to automatically fill the suggestion into the field. Be sure to grab the plugin <a href="https://github.com/Kicksend/mailcheck">here</a>!
</p>
