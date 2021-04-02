---
layout: post
title: "A gem for managing your code notes"
date: "2013-01-14"
permalink: /blog/post/a-gem-for-managing-your-code-notes
---

I just released my first gem to the world, a little command line tool for managing source code annotations. I found myself constantly scattering little notes such as TODO, OPTIMIZE, etc into my code comments, and I wanted a way to quickly view all such notes in my project directories. The result, the [Notes-CLI](https://github.com/andrewberls/notes-cli) gem, packages an executable for recursively searching through source files for common annotations with the flexibility to add your own search terms. If you've ever worked with Rails, this is very similar to the `rake notes` command, extracted to work with any project.

The gem installs the `notes` executable, which can be run with `-h` for some more information. For an example, if I run the `notes` command in one of my project directories (a Rails app), I get something similar to the following:

    $ notes app/    # Only search in the app/ directory

    app/models/user.rb:
      ln 32: # OPTIMIZE: this method can be cleaner
      ln 34: # TODO: this parameter is a hack

    app/views/expenses/_net_total.html.erb:
      ln 1: <%# TODO: condense this eventually  %>

The options for adding custom search terms or excluding directories from search are documented on the project's [Github page](https://github.com/andrewberls/notes-cli). If you want to jump right in, just run `gem install notes-cli`. 

This project started as a tiny script with a bash alias. Converting it into a gem for distribution turned out to be a trivial process, thanks to [this article](http://guides.rubygems.org/make-your-own-gem/). I've found it to be super useful, and I hope someone else is able to get some value out of it. Give it a shot and let me know what you think!