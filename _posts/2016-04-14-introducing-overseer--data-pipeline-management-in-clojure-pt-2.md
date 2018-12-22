---
layout: post
title: "Introducing Overseer - data pipeline management in Clojure (Pt. 2)"
date: "2016-04-14"
permalink: /blog/post/introducing-overseer--data-pipeline-management-in-clojure-pt-2
---

In the [last post](/blog/post/introducing-overseer--data-pipeline-management-in-clojure-pt-1), we introduced the Overseer workflow engine that ran Framed, and saw how to use plain Clojure data structures and functions to wire up an example pipeline. In this post, we'll talk about simplicity and how we were able to leverage Overseer and a variety of techniques to do away with Hadoop and other heavyweight processing frameworks.

<break />

tl;dr: We didn't have some stealth in-house distributed data processing layer. You can push flat files on S3 *amazingly* far.

That's pretty much it, actually. The reality is that most data is not that big and relatively few people truly need a system like Hadoop. This has been [written about before](https://www.chrisstucchio.com/blog/2013/hadoop_hatred.html), sometimes [humorously so](http://aadrake.com/command-line-tools-can-be-235x-faster-than-your-hadoop-cluster.html). It's really something that needs to be shouted from the rooftops, though. S3 is a truly stellar service that served us very well, and binary serialization formats (we used [Nippy](https://github.com/ptaoussanis/nippy) heavily) helped keep file sizes down and parsing snappy.

As mentioned in the last post, Overseer jobs run entirely on a single worker - even without any effort spent on compression, our job inputs fit on a 1TB local disk. Disks on AWS are not the fastest thing in the world, but this has the added benefit of skipping lots of network chatter and data transfer incurred by the distributed frameworks.

So, from an operations perspective we didn't need much. In addition to all of this, we were able to develop a set of best practices on top that kept our job code simple and testable.

## Controlling effects

Overseer doesn't have any opinions on what your jobs should do or return. However we observed that most of our jobs ran a computation over some inputs and uploaded their outputs to S3 at the end, so we developed 'artifacts', which are just data structures that specify a target to upload to S3, roughly:

<pre class="prettyprint lang-clojure"><code>{:bucket "io.framed.foobar"
 :key "/foo/bar/baz.csv"
 :file #&lt;File /tmp/...&gt;}</code></pre>

Most of our jobs would return a sequence of these artifacts, and we had a single shared generic piece of code that would actually go upload all the artifacts to S3. This is **way** more important than it might seem at first glance! It completely decouples the processing and computation steps from the actual side effects of persisting the results. Our job handlers became pure functions, for some sense of the word; they derive the required inputs they need to grab based on the job maps they're passed, then they compute a result, and return a *description* of the output value and where to put it, without mutating the state of the outside world. This is trivial to unit test like any other function! As a bonus, we only have to write and verify a single shared persistence function.

We used artifacts primarily for S3, but it's easy to imagine how the idea could be extended to broad applicability: rows to be saved in MySQL, facts in [Datomic](http://www.datomic.com/), or anything else you like. 

This is like a poor-man's [effect system](https://en.wikipedia.org/wiki/Effect_system); using simple data structures we have a first-class description of side effects, and can verify and validate them all we like before executing them. This had a tremendous impact for us, and I have a feeling it will shape lots of code in my future.

## Idempotence

All of our jobs were idempotent, meaning they could be run and re-run with no harmful side effects. This makes operations a breeze. If something goes wrong we know can always safely retry, and in fact Overseer does <strong>not</strong> go to great lengths to try and ensure exactly-once execution guarantees.

We were able to accomplish idempotence because all of our job inputs and outputs were fully deterministic and immutable. Our job handlers could compute their required inputs based on the job maps they were passed, which contained things like the company ID a report belonged to and the date the report was being generated for, etc. They would download their inputs from S3, run a pure transformation / computation over them, and return a result to be persisted. If the output already existed in S3, we could stop, or overwrite it to no ill effect.

Side note: every Overseer job has a unique ID. We used this to compute deterministic `java.util.Random` seeds per-job, so even our randomness was deterministic!

## First-class failure

Overseer provides the user a number of controls around failure handling, and maintains a few of its own. It was an important design principle for us that an unhandled exception in a job not topple the worker process - this was inspired by [Sidekiq](https://github.com/mperham/sidekiq/wiki/Error-Handling) (although we continue to debate the merits of an Erlang-style die-and-restart approach...). Overseer catches these exceptions and neatly marks the job as failed before moving on to the next job. In addition, if a job ever encounters a known bad configuration or otherwise "shouldn't happen" scenario, users can call `(overseer.api/abort)`, which will halt execution of the job *and* its dependent children. Finally, Overseer recognizes that transient errors happen; networks may be wonky, an external service may be down, etc, so `(overseer.api/fault)` will halt a job and mark it as eligible to be attempted again in the future. We configured a number of our functions to retry their operation a certain number of times before ultimately giving up and registering a fault.

These things combined meant that our Overseer workers virtually never unexpectedly fell over, with the notable exception of hard OutOfMemoryError JVM crashes from incorrectly-written job code (and ideally, a future iteration would recover even from those!). On any given day our job code could have been slow and the queue backed up, but the worker cluster kept plugging away basically no matter what and we didn't have to do much in the role of operator, especially not at 2am.

## Conclusion

A variety of techniques enabled Framed to keep data processing ridiculously simple, even at scale. Overseer was our coordination backbone that would kick off functions that in turn would download a bunch of flat binary files from S3, compute interesting results, and return a description of the result side effect: a value and a destination to upload it to. Idempotence and a first-class failure model made operations a no-brainer, and since workers didn't have to communicate with each other or a master node of any sort, we could scale our cluster up and down to match demand easily. All in all, we believe the experience was about as simple as it could be. If you're using Clojure or looking to start, we hope you give Overseer a shot. All the code is on [Github](https://github.com/framed-data/overseer), and there's a [quickstart](https://github.com/framed-data/overseer/wiki/Quickstart) and [user guide](https://www.gitbook.com/book/framed/overseer/). Happy processing!
