---
layout: post
title: "BTables: A fast, compact disk format for machine learning"
date: "2016-04-13"
permalink: /blog/post/btables-a-fast-compact-disk-format-for-machine-learning
---

*Note: This is an unmodified re-publication of a post that originally appeared on Medium.com in October 2015.*

At Framed, machine learning is our bread and butter. Every day, we run a lot of models over a lot of data, and recently ran up against some limits in the disk serialization format we were using to store our training datasets, which models use to discover patterns between input attributes and the outcome being predicted. The result is an efficient new serialization format we're excited to introduce.

<break />

The input to our models is a 2D matrix of labeled numeric features describing the activity of a given set of users. For example, we may want to record event counts for users browsing an e-commerce site. An example dataset containing three labeled features and one row per user may look like this:

<pre><code>Login  View_Cat_Food  Purchase_Cat_Food
5      3              1
2      1              0
0      0              0
10     2              2
1      0              0</code></pre>

Here, the first user logged in 5 times and viewed a listing for cat food 3 times before eventually buying some, while the second user logged in twice and viewed the product page but stopped before purchasing, and so on.

Previously, we were storing these datasets in CSV files, which are slow to parse (using the Pandas CSV parser in Python) and very inefficient in terms of disk space. This was frustrating in development, as iteration cycles were very slow, and problematic in production as model runs were consuming minutes or an hour (!), or crashing entirely as the model choked attempting to load input files that exceeded available memory. Thus we set out to find a better disk format to represent our input data, aiming to maximize both space efficiency and performance for reads/writes, as well as being cross-platform (our main backend is all Clojure, but some of our models are in Python).

We first investigated the HDF5 format, which is a cross-platform format popular for scientific datasets. Our initial investigation was not promising, as we found the code (wrapping a Java library) clunky and complex to deal with, and in some cases even slower to write or less space-efficient than an equivalent CSV! It is unclear if we wrote our prototype code in an incorrect way or didn't properly fully utilize the library, but regardless we felt the format was heavier-weight than we were looking for. HDF5 allows for modeling sophisticated arbitrary data structures/metadata using datasets in groups similar to UNIX directories/files. We knew exactly the data we need to specify, as well as the specific features we care about, and felt we could find something simpler.

First, we knew we only cared about row-by-row access over the entire file; we do not need things like random row or column reads. Second, we knew that our input data was very sparse, meaning most of the elements are zero. It seemed a trivial space optimization to only store the nonzero values, and an easy perf win to store values directly in binary to avoid continuous wasteful string parsing. So, we created BTables, a binary serialization format for sparse, labeled 2D numeric datasets (binary tables).

Conceptually, BTables are similar to a DataFrame, albeit with much fewer operations defined. As mentioned, our datasets are sparse and so we only need to store the values of nonzero cells, along with their location. We based much of our approach on the [Compressed Row Storage](http://netlib.org/linalg/html_templates/node91.html) (CRS) format, which describes arbitrarily sparse matrices with 3 vectors: <code>vals</code> for all the nonzero values (row-wise), <code>col_ind</code> for the column index of each value, and <code>row_ptr</code> denoting the indices of values in <code>vals</code> that start a row. To recount the example from the linked page, given an input matrix like

<pre><code>10 0  0  0 -2  0
3  9  0  0  0  3
0  7  8  7  0  0
3  0  8  7  5  0
0  8  0  9  9  13
0  4  0  0  2  -1</code></pre>

It would be described in CRS as:

<pre><code>vals: [10 -2 3 9 3 7 8 7 3 8 7 5 8 9 9 13 4 2 -1]
col_ind: [0 4 0 1 5 1 2 3 0 2 3 4 1 3 4 5 1 4 5]
row_ptr: [0 2 5 8 12 16 20]</code></pre>

(Note that this has been translated to 0-based indices rather than the 1-based provided in the article)

Again, the <code>row_ptr</code> corresponds to indices in the <code>vals</code> vector, so here index 0 (value = 10) begins a row, followed by index 2 (value = 3), index 5 (value = 7), and so on, with the final value containing the number of non-zeroes + 1. This is significantly more space-efficient than a CSV, and not terribly difficult to understand. We took the ideas of CRS and designed a disk layout that stores data in a single self-describing stream of values, rather than making use of multiple vectors. First, we store the length-prefixed string of labels joined together. Then, for each row we first store the number of nonzero values in that row, followed by that many (column index, value) pairs. That means the same table would be represented in a BTable with a single flat sequence (spacing for clarity):

<pre><code>2 0 10 4 -2
3 0 3 1 9 5 3
3 2 7 3 8 4 7
4 0 3 2 8 3 7 4 5
4 1 8 3 9 4 9 5 13
3 1 4 4 2 5 -1</code></pre>

This can be easily parsed in a single scan over a binary file. In addition, we can fiddle with numeric types to maximize space efficiency: all values are stored as 8-byte doubles (big-endian), and all row counts/indices are stored as 4-byte integers (big-endian).

(In fact, if we let n represent the number of rows in the matrix, CRS requires <code>(2 * num-nonzeroes) + n + 1</code> values stored to reconstruct the matrix, whereas BTables require <code>(2 * num-nonzeroes) + n</code>. Assuming equivalent numeric encodings for values/metadata, thats a massive 4 bytes of space savings over CRS! Needless to say, we went with our format because we believed it to be simpler/easier, not because were attempting to shave off every single byte we can)

The astute reader may notice that this same matrix written as a CSV only takes up 76 bytes on disk! Ignoring the additional cost of parsing values from a text file, it would seem this is a loss in terms of space efficiency. This brings us to our next point: BTables are **not** a drop-in replacement for all datasets otherwise represented as CSV! The efficiency gain is proportional to the sparsity of the dataset; for a very large, very sparse dataset, BTables are *dramatically* more space-efficient than CSVs, but for a pathological fully-dense dataset their space usage can be much worse!

Satisfied that we had designed what we believe to be a relatively simple disk format, we prototyped an initial reader/writer implementation using Java's <code>DataOutputStream</code> and <code>DataInputStream</code> classes within Clojure. This worked well and was quick to develop, but still didn't achieve the performance we were looking for. We went one level lower, and began looking into Java's NIO packages, which provide access to low-level IO operations with very high performance. We found that converting our Clojure prototype to write data to NIO buffers (containers for primitive data) and channels (connection to e.g. a file) from Java directly was a mostly painless process, and yielded a significant performance boost (in fewer than [100 lines of Java](https://github.com/framed-data/clj-btable/blob/master/src/java/io/framed/BTableWriter.java), no less).

Our current implementation is able to write a sample table with 50,000 rows and 500 features per row (25 million cells in total) in 5.8 seconds on average; the same dataset takes 16.4 seconds to write as CSV. Additionally, the CSV data files take 118 MB on disk, whereas the BTable files are only 20 MB! The gains only become more pronounced as the input scales; our production datasets may contain upwards of 1000 features per row and hundreds of thousands or millions of rows. The disk format of BTables easily allows for streaming writes and reads for datasets that are larger than available memory (this combines extremely well with lazy sequences in Clojure), but their compact representation also allows many datasets to fit in memory for model training.

Using BTables is easy! The library provides a single function to write a table to a file, and two functions to read the labels or rows from a given table file. Here's an example in Clojure:

<script src="https://gist.github.com/andrewberls/27bbeedc76d7db75d4c9.js"></script>

No <code>HDF5IntStorageFeatures$HDF5IntStorageFeatureBuilder</code> in sight here.

Right now, we have open-source libraries available for [Clojure](https://github.com/framed-data/clj-btable) and [Python](https://github.com/framed-data/btable-py) (we've even written an experimental [Haskell](https://github.com/andrewberls/btable-haskell) binding for fun), with documentation and examples available in each repository. We've been successfully running BTables in production for just over six months and were excited to share our work with the community! If you're working with sparse datasets we hope you give BTables a shot!
