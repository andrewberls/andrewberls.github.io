---
layout: post
title: "A Rakefile to build CoffeeScript projects"
date: "2012-07-15"
permalink: /blog/post/rakefile-to-build-coffeescript-projects
---

<em style="font-size:1.1em">Update: I've since overhauled this project to be an npm package called Arabica. You should [check it out.](https://github.com/andrewberls/arabica)</em>

<p>
I recently started a new project in CoffeeScript, and began by looking for a good build system. When developing a JavaScript-heavy application, concatenating and minifying scripts is crucial in order to reduce the number of HTTP requests and download size, making your app nice and snappy. Unfortunately order can matter when concatenating files, so randomly lumping together everything in the source directory isn't enough. I set out to find a solution that allowed me to specify files in the order to compile, with the ability to watch a directory for convenience. This is what I threw together - a Ruby Rakefile that makes use of the uglifier and listen gems, inspired by a script by <a href="https://github.com/claydotio/Slime-Volley/blob/master/Rakefile">Clay.io</a>.
</p>

<break />

<script src="http://gist.github.com/3118851.js"></script>

<h3>Usage</h3>
<p>
Using the script is very simple, but there are a few dependencies that must be resolved beforehand. An installation of <a href="http://www.ruby-lang.org/en/downloads/">Ruby</a> and <a href="http://rubygems.org/pages/download/">RubyGems</a> is required. The Gemfile is only present for convenience - the script should work just fine if happen to already have the gems installed or want to install them manually, but you'll want the Rakefile in your project's root. If you're installing using the Gemfile, grab bundler if necessary with <code>gem install bundler</code> and finally <code>bundle install</code> to install the uglifier and listen gems. Once you've set up all dependencies, put your project filenames (in the desired order) in the <code>FILENAMES</code> array, and customize the <code>SRC_PATH</code>, <code>BUILD_PATH</code>, and <code>DIST_PATH</code> variables to your needs. You can call <code>rake build</code> manually to build the application, or just watch the source files with <code>rake watch</code> which will compile automatically anytime you save a change.
</p>

<h3>Notes</h3>
<p>
I tend to keep my editor and terminal on separate workspaces, so I added a <a href="http://growl.info/">Growl notification</a> that alerts you if a compile attempt failed - this is totally optional and can be removed if you prefer. Finally, the <code>rake clean</code> task removes all compiled files (including distribution files) if you ever need to clean up your directory.
</p>

<p>
I've already run into the issue trying to include non-CoffeeScript files (such as vendor JS libraries), but it's entirely possible with a small hack and I can update to post that if anyone would like. As always, I welcome ideas or suggestions for improvement!
</p>