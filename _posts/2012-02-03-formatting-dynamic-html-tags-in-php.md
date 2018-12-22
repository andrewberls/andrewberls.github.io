---
layout: post
title: "Easily formatting dynamic HTML tags in PHP"
date: "2012-02-03"
permalink: /blog/post/formatting-dynamic-html-tags-in-php
---

<p>
It's probably somewhat ironic that I'm writing my first code-oriented post about PHP. After all, I'm becoming more of a Rails fanboy the more I learn about it, and putting down PHP is one of my favorite pastimes. That being said, we're about to roll out a Drupal installation over at <a href="http://www.soc.ucsb.edu/sexinfo/">SexInfo Online</a>, and so at least a cursory knowledge of PHP is required.
</p>





<h5>So what are we doing here?</h5>
<p>I got sick of seeing code that looked like this:</p>

<pre><code>&lt;?php
  while($row = mysql_fetch_array($result)){
    echo "&lt;div class=\"item\"&gt;" . $row['field']; . "&lt;/div&gt;";
  }
?&gt;
</code></pre>

<p>I decided that I needed a utility function similar to Rails' own <a href="http://api.rubyonrails.org/classes/ActionView/Helpers/TagHelper.html#method-i-content_tag">content_tag</a> method, which allows you to do something like this:</p>

<pre><code>&lt;%= content_tag(:p, "Hello World!", :class => "strong") %&gt;
</code></pre>

<p>The function is taking 3 parameters in this case: the type of tag (as a Ruby symbol), the content itself, and optional HTML attributes such as classes or an ID. So let's model our PHP version on this, and write down some sample usage.</p>

<pre><code><span class="label">// Our end goal:</span>
&lt;?php
  $content = "Hello World!";
  echo content_tag("p", $content, array("class" => "strong"));
  <span class="comment">// Should return &lt;p class="strong"&gt;Hello World!&lt;/p&gt;</span>
?&gt;
</code></pre>

<p>Let's go about actually implementing this!</p>

<break />


<h5>Getting Started</h5>
<p>Our first step is to think about the parameters that our <code>content_tag()</code> method will accept, but we've already covered that. We need to be able to specify the type of tag, the content, and optional attributes. Let's write our basic skeleton.</p>

<pre><code>&lt;?php
  function content_tag($tagName, $content, $attr=array()) {
    <span class="comment">// Implementation later!</span>
  }
?&gt;
</code></pre>

<p>Nothing too complicated here. The <code>$tagName</code> and <code>$content</code> variables will be passed in as strings. Attributes are slightly more tricky; we're giving them a default value of a blank array, and will handle them as key-value pairs when used, e.g., <code>"class" => "strong"</code></p>


<p>Before we move on and start writing code, let's think about what an HTML tag actually consists of. Take a look at this simple tag:</p>

<pre><code>&lt;p&gt;Hello World!&lt;/p&gt;
</code></pre>

<p>This is about as simple as it gets. It consists of a the opening tag (<code>&lt;p&gt;</code>), the content, and the closing tag(<code>&lt;/p&gt;</code>). Already some helper methods are becoming apparent: one each for the opening and closing tag. The methods will be very similar: each one needs to take in a <code>$tag</code> parameter so that the appropriate tag can be output, but the opening tag also needs to account for possible HTML attributes. Let's start backwards and write the <code>close_tag()</code> method first.</p>




<h5>The close_tag() method</h5>
<p>This is simplest helper method we'll be writing. All we need to pass in is the type of tag to be closed!</p>

<pre><code>
&lt;?php
  function close_tag($tag) {		
    return "&lt;/" . $tag . "&gt;";
  }
?&gt;
</code></pre>

<p>Not a whole lot to say there. Onwards!</p>




<h5>The open_tag() method.</h5>
<p>Our job gets a little bit trickier here. The tag itself is very simple, similar to <code>close_tag()</code>, except that we need to account for HTML attributes as a possible parameter. As I mentioned before, attributes will be handled as key-value pairs in an array. To translate this into HTML, we need to loop through all the pairs in our attribute array and output it as <code>key="value"</code>. Let's put this in a helper method to prevent our <code>open_tag()</code> method from getting cluttered, and call it <code>generate_attr_string()</code>, a function which will take a single array parameter.</p>

<pre><code>&lt;?php
  <span class="comment">// Example usage: generate_attr_string(array("id" => "nav", "class" => "item"));</span>
  <span class="comment">//=> id="nav" class="item"</span>
  
  function generate_attr_string($attr) {
    $attr_string = null;
    if (!empty($attr)) {
      foreach ($attr as $key => $value) {
        <span class="comment">// If we have attributes, loop through the key/value pairs passed in</span>
        <span class="comment">//and return result HTML as a string</span>
        
        <span class="comment">// Don't put a space after the last value</span>
        if ($value == end($attr)) {
          $attr_string .= $key . "=" . '"' . $value . '"';
        } else {
          $attr_string .= $key . "=" . '"' . $value . '" ';
        }
      }
    }		
  return $attr_string;		
}
?&gt;
</code></pre>

<p>That's a little ugly, but it makes our <code>open_tag()</code> method trivial to finish. Let's wrap that up.</p>

<pre><code>&lt;?php
  <span class="comment">// Example usage: open_tag("p", array("id" => "nav", "class" => "item"));</span>
  <span class="comment">//=> &lt;p id="nav" class="item"&gt;</span>
  
  function open_tag() {
    $attr_string = generate_attr_string($attr);
    return "&lt;" . $tag . " " . $attr_string . "&gt;"; 
  }
?&gt;
</code></pre>

<p>Starting to see how <code>content_tag()</code> is going to work? Stay tuned - we're almost there!</p>

<h5>The Final Function</h5>
<p>At this point we're really just stringing together our utility methods, using our parameter signature from before. Let's see how it looks!</p>
<pre><code>&lt;?php
  function content_tag($tagName, $content, $attr=array()) {
    return open_tag($tagName, $attr) . $content . close_tag($tagName);
  }
?&gt;
</code></pre>

<p>It's almost a disappointing finish - all the fun was in the helper methods! You could put keep all of the code in the <code>content_tag()</code> method if you wanted to, but I generally try to keep everything separated out into functions. Your choice. Anyways, I hope that you find this function useful one day - the use case I described in the beginning of this post was trivial, but I find this comes in handy when looping through results returned by MySQL or similar.</p>

<p>Anyways, that's hopefully the last I'll speak of PHP for a while. I promise the next code post will be delicious Rails :)</p>

