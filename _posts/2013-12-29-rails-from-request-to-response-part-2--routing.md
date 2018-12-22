---
layout: post
title: "Rails from Request to Response: Part 2 - Routing"
date: "2013-12-29"
permalink: /blog/post/rails-from-request-to-response-part-2--routing
---

This is part 2 of a series taking an in-depth look at how the internals of Rails handle requests and produce responses - you can read part 1 [here](/blog/post/rails-from-request-to-response-part-1--introduction).

Last time we took a brief look at how Unicorn workers accept clients from a shared socket and call our Rails application, and how requests bubble down through the middleware stack before arriving at `Blog::Application.routes`. We're now at the routing stage, where we need to match the request URL to a controller action and invoke it to get a response.

<break />

As mentioned, the `#call` method will be invoked on whatever the `Blog::Application.routes` method returns (as it happens, most of this post is actually just tracing the `#call` method to the point where we get a response).

The definition of `Blog::Application.routes` is located in the same engine.rb file:

<pre class="prettyprint lang-ruby"><code># rails/railties/lib/rails/engine.rb<br />
def routes
  @routes ||= ActionDispatch::Routing::RouteSet.new
  @routes.append(&Proc.new) if block_given?
  @routes
end</code></pre>

So `Blog::Application.routes` returns an instance of `ActionDispatch::Routing::RouteSet`. This is the first reference we'll see to ActionDispatch, which is a module that's part of ActionPack. ActionDispatch handles tasks such as routing, parameter parsing, cookies, and sessions. ActionPack is one of the top-level gems in the Rails source code, and encompasses the 'VC' in Rails MVC - it handles routing as mentioned previously, as well as controller definitions (ActionController) and view rendering (ActionView). The `RouteSet` class appears to be the pairing of a table of routes with a router to interpret and dispatch requests to controller actions - we'll see more on these parts later.

Back to our engine - we need to examine the `#call` method for the `RouteSet`. The code lives in `actionpack/lib/action_dispatch/routing/route_set.rb`. Here's the relevant code:

<pre class="prettyprint lang-ruby"><code># rails/actionpack/lib/action_dispatch/routing/route_set.rb<br />
module ActionDispatch
  module Routing
    class RouteSet<br />
      def call(env)
        @router.call(env)
      end<br />
      ...<br />
    end
  end
end</code></pre>

So the RouteSet hands things off to whatever's in `@router`. If we look at the [RouteSet constructor](https://github.com/rails/rails/blob/0dea33f770305f32ed7476f520f7c1ff17434fdc/actionpack/lib/action_dispatch/routing/route_set.rb#L299), we can see that `@router` is an instance of `Journey::Router`.


## Journey

Journey is the core routing module in ActionDispatch. Its [codebase](https://github.com/rails/rails/tree/0dea33f770305f32ed7476f520f7c1ff17434fdc/actionpack/lib/action_dispatch/journey) is fascinating to browse as it actually uses a generalized transition graph (GTG) and non-deterministic finite automata (NFA) to match URLs and routes. Pull out those theoretical CS textbooks! Journey even includes a full-blown [yacc grammar file](https://github.com/rails/rails/blob/0dea33f770305f32ed7476f520f7c1ff17434fdc/actionpack/lib/action_dispatch/journey/parser.y) for parsing routes! I'll be writing more about Journey in future posts.

The `#call` method in `Journey::Router` is a bit lengthy, but here are the relevant bits:

<pre class="prettyprint lang-ruby"><code># rails/actionpack/lib/action_dispatch/journey/router.rb<br />
module ActionDispatch
  module Journey
    class Router<br />
      def call(env)
        find_routes(env).each do |match, parameters, route|
          env[@params_key] = (set_params || {}).merge parameters
          status, headers, body = route.app.call(env)
          return [status, headers, body]
        end
      end<br />
      ...<br />
    end
  end
end</code></pre>

This is the same Rack interface we saw in the Unicorn `#process_client` code in part 1 - this is the heart of Rails, and all other Rack-compatible frameworks.


`#find_routes` runs the URL through the GTG simulator using the routing table - it'll take a separate post to explain the mechanism, but as its name suggests it returns a list of routes that match the requested URL. The routing table contains all of the routes of the system (instances of `Journey::Route`), and is itself an instance of `Journey::Routes`, which is constructed by the code you define in `config/routes.rb`. The construction of the routing table and the `ActionDispatch::Routing::Mapper` class could fill a post alone, and I highly recommend RailsCasts [#231](http://railscasts.com/episodes/231-routing-walkthrough) and [#232](http://railscasts.com/episodes/232-routing-walkthrough-part-2) for anyone looking for a detailed look at route construction.

Once we've found a matching route, we call `#call` on the app associated with the route. Now, whenever we see a reference to `app` in the Rails source, it's a safe bet to assume that it refers to a Rack app. As it turns out, controller actions act like Rack apps! We'll see more on this later on.

Tracking down where the app associated with a route gets initialized is a little tricky. The short of it is that when constructing the routing table, the `ActionDispatch::Routing::Mapper` class calls `add_route`, which will construct a `Journey::Route` instance associated with a controller endpoint (the Rack app) and add it to the routing table. However, the class definition in `mapper.rb` is almost  is almost 2000 lines! Here's the shortened definition of `Mapper#add_route`:

<pre class="prettyprint lang-ruby"><code># rails/actionpack/lib/action_dispatch/routing/mapper.rb<br />
module ActionDispatch
  module Routing
    class Mapper<br />
      def add_route(action, options)
        path = path_for_action(action, options.delete(:path))<br />
        ...<br />
        mapping = Mapping.new(@set, @scope, URI.parser.escape(path), options)
        app, conditions, requirements, defaults, as, anchor = mapping.to_route
        @set.add_route(app, conditions, requirements, defaults, as, anchor)
      end<br />
      ...<br />
    end
  end
end</code></pre>

Here, `@set` is the `RouteSet` we were dealing with previously, and a little digging reveals that the `app` referenced here is an instance of `ActionDispatch::Routing::Dispatcher`. The `Dispatcher` class is defined back in `route_set.rb` - here's its (simplified) definition of `#call`. To make things more clear, imagine that we're browsing to the `/posts` URL of our blog app, which routes to the index action on the Posts controller.

<pre class="prettyprint lang-ruby"><code># rails/actionpack/lib/action_dispatch/routing/route_set.rb<br />
module ActionDispatch
  module Routing
    class Dispatcher<br />
      def call(env)
        params = env[PARAMETERS_KEY]
        prepare_params!(params) 
        # params = {:action=>"index", :controller=>"posts"}<br />
        controller = controller(params, @defaults.key?(:controller))
        # controller = PostsController<br />
        dispatch(controller, params[:action], env)
      end<br />
      ...<br />
    end
  end
end</code></pre>


We're getting to the good stuff! At this point the `controller` method has taken the interpreted controller parameter (such as `"posts"`) and given us a reference to the class (`PostsController`), and we have a basic `params` hash. We're ready to call the action on the controller and get our full-blown response - here's the definition of the `dispatch` method:

<pre class="prettyprint lang-ruby"><code># rails/actionpack/lib/action_dispatch/routing/route_set.rb<br />  
def dispatch(controller, action, env)
  controller.action(action).call(env)
end</code></pre>

At this point we've successfully matched the route to a controller/action, and the request is sent off into the ActionController stack, which will be the subject of the next post. Take a deep breath - routing is a particularly complex topic within Rails and the method chain can get pretty huge. Of course, as mentioned it's not necessary to fully understand every detail of Rails routing. Part of the magic of Rails is that it hides all of these messy details from you! However, I find it fascinating to trace through and see the full flow of execution.

Some key takeaways from this post are the role of ActionPack in handling web requests from start to finish, and how controller actions behave like simple Rack endpoints. You can test this one for yourself by opening up `rails console` and running something like:

<pre><code>2.0.0-p247 :001> PostsController.action("index")
 => #&lt;Proc:0x007fa9fd356d40@rails/actionpack/lib/action_controller/metal.rb:269&gt;</code></pre>

If you were to `#call` that with the proper `env` hash, you'd get the usual array of `[status,headers,body]`!

In the next post, we'll dive into ActionController, metal and all. Until next time!


<div class="series-next-container">
<span>Ready for more? Read <strong>Part 3 - ActionController</strong> 
<span class="arrow">→</span></span>
<a class="btn btn-blue" href="/blog/post/rails-from-request-to-response-part-3--actioncontroller">Go!</a>
</div>