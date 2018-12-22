---
layout: post
title: "API Versioning with Rails 4"
date: "2013-12-13"
permalink: /blog/post/api-versioning-with-rails-4
---

I recently started a new side project with Rails 4 that's primarily based on an API. I wanted to get API versioning right from the start, and spent longer than I'd like wrestling with Rails routing. This post details the approach I ended up using in the end.

### The Goal

Here's an example of the type of route I was hoping to achieve:

```
POST http://api.mysite.com/v1/events/
```

Using a subdomain instead of something like `/api/v1/events/` is just a preference, although both are easy to accomplish in Rails. The next part is tricky however. I wanted a directory structure that looks like something this:

```
app/controllers/
.
|-- api
|   `-- v1
|       |-- api_controller.rb
|       `-- events_controller.rb
|-- application_controller.rb
```

Here, `ApiController` is a subclass of `ApplicationController`, and acts as a parent class for all of the other API controllers. This lets us keep code for generic API stuff like access checking and resource delivery in one place (Zach Holman has some great notes about this in [Ruby Patterns from GitHub's Codebase](http://zachholman.com/talk/ruby-patterns/)).

<break />

### The Result

Here's the final code for the routing:

<pre class='prettyprint lang-ruby'><code>constraints subdomain: 'api' do
  scope module: 'api' do
    namespace :v1 do

      resources :events

    end
  end
end
</code></pre>

The `scope module: 'api'` bit lets us route to controllers in the `API` module without explicitly including it in the URL. However, the version (`v1/`) is part of the URL, and we also want to route to the `V1` module, so we use `namespace`. These two things combined means that any controllers we define must be nested within the `Api::V1` namespace.

Here's what the controllers look like:

<pre class='prettyprint lang-ruby'><code># app/controllers/api/v1/api_controller.rb

module Api::V1
  class ApiController < ApplicationController
    # Generic API stuff here
  end
end
</code></pre>

<pre class='prettyprint lang-ruby'><code># app/controllers/api/v1/events_controller.rb

module Api::V1
  class EventsController < ApiController

    # POST /v1/events
    def create
      render json: params.to_json
    end

  end
end
</code></pre>



And we're done! We have nicely versioned and separated API code and we can POST away to our heart's content.

```
curl --data "hello=world" http://api.mysite.com/v1/events
# => {"hello":"world","subdomain":"api","action":"events","controller":"api/v1/events"}
```

Implementing a sweet API is left as an exercise for the reader.


Anyways, this is my take on API versioning. The project is still in its infancy and I'm still a novice to this whole thing. One downside I can see is that if/when the API version gets bumped, a lot of code would likely need to be duplicated, but arguably this is one use case where that might be acceptable. I'd love to hear any different opinions on how it should be done! 