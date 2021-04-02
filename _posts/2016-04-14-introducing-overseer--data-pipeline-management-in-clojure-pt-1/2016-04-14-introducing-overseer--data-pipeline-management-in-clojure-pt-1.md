---
layout: post
title: "Introducing Overseer - data pipeline management in Clojure (Pt. 1)"
date: "2016-04-14"
permalink: /blog/post/introducing-overseer--data-pipeline-management-in-clojure-pt-1
---

Overseer is a library for building and running data pipelines in Clojure. Workflows are designed as a graph ([DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph)) of dependent tasks, and it handles scheduling, execution, and failure handling among a pool of workers. 
It ran the vast majority of the [Framed predictive analytics platform](https://angel.co/framed-data) for over a year up to the [acquisition by Square](http://techcrunch.com/2016/03/14/square-brings-on-the-team-behind-framed-data-a-predictive-analytics-startup/), and today we're happy to release the project as open source to the community.

<break />

In simpler words, Overseer lets you run a bunch of batch jobs that have dependencies on each other, i.e. a given job can't run until all the jobs it depends on have completed successfully. This has broad applications for ETL, data ingestion, building complex reports, and more. At Framed, we used it for a wide variety of tasks including pulling data from external providers, computing aggregations and metrics, generating machine learning training sets, running models, and parsing/persisting their output.

This is the first post in a two-part series. In this post, we'll explain the rationale for building Overseer and a bit about mechanics - how the system works, key terms, and what it's like to operate in production. In the next post, we'll discuss principles and best practices that shaped our approach to simplifying data crunching at scale.

Overseer operates in a crowded space of so-called workflow management engines, and is conceptually similar to [Azkaban](https://github.com/azkaban/azkaban), [Airflow](https://github.com/airbnb/airflow), and [Luigi](https://github.com/spotify/luigi). From the respective descriptions:

<blockquote><p>Azkaban [LinkedIn] is a batch workflow job scheduler created at LinkedIn to run Hadoop jobs. Azkaban resolves the ordering through job dependencies and provides an easy to use web user interface to maintain and track your workflows.</p></blockquote>

<blockquote><p>Airflow [AirBnB] is a system to programmatically author, schedule and monitor data pipelines.</p></blockquote>

<blockquote><p>Luigi [Spotify] is a Python module that helps you build complex pipelines of batch jobs. It handles dependency resolution, workflow management, visualization etc.It also comes with Hadoop support built in.</p></blockquote>

We've also seen mention of similar projects at [Medium](https://medium.com/medium-eng/the-stack-that-helped-medium-drive-2-6-millennia-of-reading-time-e56801f7c492#.zgvrcofgd), [IFTTT](http://engineering.ifttt.com/data/2015/10/14/data-infrastructure/), and the [McDonnell Genome Institute](https://github.com/genome/ptero), and surely more exist. So why in the world did we develop our own solution?

Our primary reason for developing something new was simplicity. Framed did not use Hadoop or any of the related ecosystem projects - we believed they were heavy, complex tools, and the operations side is extremely daunting for a very small team lacking lots of ops experience. Thus, the existing workflow managers that plugged into Hadoop weren't a good fit for our needs. Several other solutions were written in Python, and we preferred to remain in Clojure or at least on the JVM. The result is that Overseer is written in Clojure for Clojure, and is not tied to Hadoop or any other ecosystem. 

From a design standpoint, Overseer favors ordinary Clojure data structures and functions over all else. We didn't want to to have to twist our code to match a particular framework, so no special classes or `InputFormats` are required here. Overseer imposes very little structure and overhead - jobs and their dependencies are specified in plain Clojure maps, and handlers are ordinary functions that can be run locally and tested like any other; these are incredibly important aspects for keeping things simple and reasoning about the system.

## Running Overseer

From an operational standpoint, Overseer keeps track of all jobs in [Datomic](http://www.datomic.com/), which is a fantastic database from the same people who developed Clojure, and reflects many of the same values. Jobs in Overseer are durable; they are not de-queued or deleted after they complete, meaning the framework operates in a slightly different space than background job processing systems like [Sidekiq](http://sidekiq.org/). Overseer integrates into application code as library, and we tend to call each node running an instance of the library a worker. Each worker will continuously resolve dependencies to find an eligible job, run it, and track its status, gracefully handling failures. Overseer is a masterless system, i.e. every worker has an identical role, and clusters can be seamlessly scaled out horizontally. Framed's primary cluster elastically varied from fewer than 10 to dozens of nodes and Overseer never knew the difference.

## A brief example

So what does Overseer look like? Spoiler: most of this code is lifted from our [quickstart](https://github.com/framed-data/overseer/wiki/Quickstart) and there's an extensive [user guide](https://www.gitbook.com/book/framed/overseer/).

Here's an entire example namespace that defines a job graph and fires up a worker:

<pre class="prettyprint lang-clojure"><code>(ns myapp.core
  (:require [overseer.api :as overseer])
  (:gen-class))

(def job-graph
  {:start []
   :result1 [:start]
   :result2 [:start]
   :finish [:result1 :result2]})

(def job-handlers
  {:start (fn [job] (println "start"))
   :result1 (fn [job] (println "result1"))
   :result2 (fn [job] (println "result2"))
   :finish (fn [job] (println "finish"))})

(defn -main [& args]
  (let [config {:datomic {:uri "datomic:free://localhost:4334/myapp"}}]
    (overseer/start config job-handlers)))</code></pre>

There are a few important components at play here. First is `job-graph`: this is an ordinary Clojure map that abstractly describes the types of jobs and dependencies between them. Each job describes the jobs it depends on, i.e. its "parents" (this may be somewhat the reverse of other graph notations where each node describes its children, i.e. all arrows going downwards). Here the `:start` job type has no dependencies, and so is eligible for execution as soon as we create an instance to be run. The `:result1` and `:result2` jobs both depend on `:start`, and they will not run until the `:start` job successfully completes. Similarly, the `:finish` job depends on both `:result1` and `:result2`. Overseer handles scheduling and execution, so if any job fails unexpectedly, its dependent children will not run.

There is some opportunity for magic behind the scenes - since `:result1` and `:result2` do not depend on each other, as soon as the `:start` job finishes, both `:result1` and `:result2` may start executing immediately in parallel on different machines in your cluster.

The next important concept is `job-handlers`. This is a Clojure map where the keys are job types corresponding to the dependency graph from before, and the values are ordinary Clojure functions to run. Overseer will automatically call these functions and pass in a `job` argument, which is a map of information about the current job. These examples just print a message to stdout and don't do anything meaningful; real jobs of course will likely perform computation and persist their results to external storage, enabling data dependencies between jobs.

## What Overseer is not

Overseer is not a distributed data processing library like Spark or MapReduce. Each job runs on a single worker, meaning the practical limit on input size is the worker's disk size (you can push a terabyte surprisingly far!), and workers have no communication with each other. Nor does it even run on multiple CPU cores per-worker, although it could choose to easily; instead it leaves it to each job to maximally utilize all the cores and resources available. These decisions and tradeoffs worked extremely well for our use cases, but it's not for everyone; the project doesn't aim to be the one true solution for every situation.

As a side note, we wrote all of our jobs as native Clojure functions, but in theory if one so desired Overseer could be used purely as a thin coordination layer and shell out a process for code written in some other language or system (e.g. a Spark job)

## Wrapping up

To recap, Overseer was the backbone of the Framed processing pipeline for over a year, and ran hundreds of thousands of jobs during that time. Its focus on simplicity made it trivial for us to define new jobs, modify an existing dependency graph, test job code thoroughly, and scale up with more machines. In the [next post](/blog/post/introducing-overseer--data-pipeline-management-in-clojure-pt-2) we'll dive into how we were able to avoid Hadoop and other heavyweight frameworks and keep our jobs simple at scale. In the meantime, be sure to check out the code on [Github](https://github.com/framed-data/overseer), or try the [quickstart](https://github.com/framed-data/overseer/wiki/Quickstart) and [user guide](https://www.gitbook.com/book/framed/overseer/).

<div class="series-next-container">
<span>Ready for more? Read <strong>Part 2</strong> 
<span class="arrow">→</span></span>
<a class="btn btn-blue" href="/blog/post/introducing-overseer--data-pipeline-management-in-clojure-pt-2">Go!</a>
</div>