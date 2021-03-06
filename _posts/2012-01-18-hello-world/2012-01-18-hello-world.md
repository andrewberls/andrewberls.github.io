---
layout: post
title: "Hello World!"
date: "2012-01-18"
permalink: /blog/post/hello-world
---

<p>Even though this is the obligatory 'Hello World' post, it feels like it's been forever since I started work on planning and building this site. In early fall of last year, I was gifted the domain andrewberls.com. I had no particular use for it at the time, but it seemed like a handy thing to hold on to. I decided I wanted to build a portfolio site for myself, to show off a little of what I've done up until this point and shamelessly promote myself. After completing the first iteration while also frantically trying to juggle schoolwork, I decided that the site needed more to it. Everybody has a blog, and I wasn't going to be the exception. I pondered my options- I posted on tumblr for the time being and thought about throwing a Wordpal or Drupal installation onto my domain, but that seemed unexciting. Nobody cares if you can install Wordpress, and I wanted to focus on improving my technical knowledge and abilities rather than actual writing (god forbid) anyways. The result you see here is powered by Sutro, a blogging engine I wrote from scratch using Ruby on Rails.</p>

<p>Going into all of this, I had absolutely no clue what Rails was all about, and had never used most of the tools I needed in any significant application. I had started learning a few snippets of PHP towards the end of the summer, and so the initial application was written in PHP. I started on the portfolio section, and only needed some basic scripting for the Contact form, so it seemed like a good fit. When I decided to write the blogging software as well, it was pretty clear that I would need a better solution, and so I had a good excuse to go learn Rails.</p>



<p>In building this site, I've learned more than I could have possibly imagined. The various features I chose to initially implement forced me to finally learn Ruby and what Rails was all about, the  model-view-controller architecture, databases, authentication algorithms, optimization techniques, and some sneaky JavaScript tricks with a healthy dose of jQuery. It's not to say that I didn't know anything about these beforehand, but I'm a firm believer in that the only way to truly learn something is to get your hands dirty and work with it. I decided fairly early on that I wanted to avoid third-party plugins and pre-written code as much as possible. It's not because I don't see their value- to the contrary, I'm an advocate of letting other people do the heavy lifting for you, and believe it to be a best practice - but I wanted to make this a learning experience as much as possible for myself and see what I could accomplish from scratch. Some of the resulting features - slideshows, contact forms, etc, are a little rough around the edges, but they work and I get to say that it's something I wrote.</p>

<break />

<h5>Behind the Scenes</h5>
<p>The front end of the site is fairly straightforward.  I use the <a href="http://code.google.com/p/zen-coding/">Zen Coding</a> plugin with <a href="http://www.aptana.com/">Aptana Studio</a> as my primary editor environment. All of the CSS is processed through <a href="http://sass-lang.com/">Sass</a>, and most of the JavaScript is handled by jQuery. None of these particular tools are critical to the project or its structure, but they make life much easier, and save heaps of time. On the back end, I'm running Ruby on Rails with PostgreSQL deployed to Heroku (with Zerigo DNS to route back to BlueHost), which is a result of many, many hours of tearing my hair out over obscure bugs and errors. All of the source code is managed by Git and hosted on <a href="https://github.com/andrewberls/andrewberls">GitHub</a>. I'm still new to using Git, and so the freely-available <a href="http://progit.org/book/">Pro Git</a> has been invaluable in helping me get off the ground. As much as I wanted to avoid plugins, I had to cave on several things where it's simply too inconvenient to avoid. I use BCrypt for password hashing and salt generation, and <a href="https://github.com/mislav/will_paginate/wiki">will_paginate</a> for pagination of the post display (which is awesome for simple applications). What you see here is only half of the end result - everything is supported by a custom-built backend providing administrative views and actions on posts, as well as primitive support for multiple users with permission levels.</p>




<h5>Design</h5>
<p>I've gone back and forth on the design of the site. From the beginning, I knew I wanted a single-page scrolling layout for the front page, and a strong focus on clean, minimalist design. As it turns out, I'm terrible at design and coming up with something attractive and professional-looking is difficult to do. I constantly wavered on color palettes and layouts, and ended up changing the look of the blog at the very last minute. The design is heavily influenced by Twitter's <a href="http://twitter.github.com/bootstrap/">Bootstrap</a> framework which is absolutely brilliant, and I'll definitely be using it on my next project. The two tools I can no longer live without are <a href="http://colorschemedesigner.com/">Color Scheme Designer</a> and Eric Meyer's <a href="http://meyerweb.com/eric/tools/color-blend/">Color Blender</a>.</p>




<h5>Looking Back</h5>
<p>This may seem like a fairly trivial application in the end. Blogging software is actually relatively simple conceptually to write: a couple of forms POST to controllers which save the data and essentially spit it back out to the views for display. That's a generalization of course: the tricky bits are all in the tiny details in getting the bigger concepts to work, and I consider it a personal accomplishment. This is one of the first significant "real" projects I've worked on, and something I completed of my own volition. It's been fascinating to watch the project evolve, even from my own viewpoint. Since everything is hosted on GitHub, I can go back in time through the commit history and look at day one, where it was a single page written in PHP, up through where I first migrated to Rails (and wrote some pretty nasty code), and finally up to where the project is now.</p>




<h5>The Future</h5>
<p>So what's next? The site is not finished it any sense of the word. I have the ability to write and administer posts, which was my initial goal, but I plan to continue to expand the project in my free time. I only just touched on multi-user CMS-style support, and I have no shortage of refactoring and cleaning up to do (past me is a real slob). I have plenty of other projects on my plate as well. I'll be rolling out a complete rewrite of <a href="http://www.soc.ucsb.edu/sexinfo/">SexInfo Online</a> in the weeks to come. I'm also just starting to look more closely into JavaScript and the HTML5 canvas element as part of writing a scrolling browser game for a class. I'll be writing posts about all of these in the future, as well as more in-depth looks at some of the various Sutro feature implementations with tips and tricks.</p>



<h5>Goodbye World (for now)</h5>
<p>If you're interested, be sure to check out the nitty gritty implementation over on <a href="https://github.com/andrewberls/andrewberls">GitHub</a>, or <a href="http://andrewberls.com/#contact">shoot me an email</a>!</p>

<p>Oh, and the best part? This entire thing was developed on a Windows environment. How's that for a healthy dose of sadomasochism? ;)</p>
