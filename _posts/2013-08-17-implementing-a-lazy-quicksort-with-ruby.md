---
layout: post
title: "Implementing a lazy quicksort with Ruby"
description: ""
category: 
tags: []
---
{% include JB/setup %}

![Quicksort example](/assets/quicksort.gif)

While working through Michael Fogus' excellent book [The Joy of
Clojure](http://joyofclojure.com/), I came across his implementation of the
[quicksort algorithm](http://en.wikipedia.org/wiki/Quicksort) and was
determined to unpack the implementation. Not being super familiar with
quicksort in general, and still unfamiliar with a lot of Clojure idioms, I
thought that reimplementing the algorithm in Ruby would be a good way to force
myself to learn the nuances of the implementation and better appreciate what
Clojure brings to the table.

For reference, here's the full Clojure implementation:

<pre><code class='brush: Clojure'>
(defn nom [n] (take n (repeatedly #(rand-int n))))

(defn sort-parts
  "Lazy, tail-recursive, incremental quicksort. Works against
  and creates partition based on the pivot, defined as 'work'."
  [work]
  (lazy-seq
    (loop [[part & parts] work]
      (if-let [[pivot & xs] (seq part)]
        (let [smaller? #(< % pivot)]
          (recur (list*
                   (filter smaller? xs)
                   pivot
                   (remove smaller? xs)
                   parts)))
        (when-let [[x & parts] parts]
          (cons x (sort-parts parts)))))))

(defn qsort [xs]
  (sort-parts (list xs)))
</code></pre>

To get started, let's define a method which can give us our supply of random
integers to feed into our sorter. The normal way to do this in Ruby would be
with an iterator:

<pre><code class='brush: ruby'>
def nom(n)
  n.times.map { rand(1000) }
end
</code></pre>

In the Clojure example however, the numbers are pulled out of an lazy infinite
sequence. We can reproduce that in Ruby using an
[Enumerator](http://ruby-doc.org/core-2.0/Enumerator.html). If you aren't too
familiar with these, I highly recommend reading through Gregory Brown's
tutorial on [building Enumerable & Enumerator from
scratch](https://practicingruby.com/articles/shared/eislpkhxolnr).

<pre><code class='brush: ruby'>
def nom(n)
  Enumerator.new { |yielder|
    loop { yielder.yield(rand(1000)) }
  }.take(n)
end
</code></pre>

The Enumerator holds inside of itself a loop, and when it is called upon to
give `n` objects by `take`, it runs the loop that many times to return from the
method `n` random numbers. In practice these two methods return the same
results, but wrapping your head around this simple usage of Enumerator will pay
off once we get into the more complicated usage in the next part.

The first step in quicksort is to pick a number from a list of numbers. This
element is called the 'pivot'. You then proceed to reorder the list so that
smaller numbers are moved before the pivot, and larger numbers are moved after
it. Then you recursively apply the same algorithm to the smaller list and the
larger list, until you have sorted each sub-list.

The trick used in the Clojure example is that it holds off on sorting the
larger lists until it runs out of smaller ones, in order to be able to start
returning results without having to sort everything.  This is what laziness is
all about: focusing on getting the first result, while procrastinating on as
much of the other work as we can.

My attempt at converting the Clojure example as closely as possible:

<pre><code class='brush: ruby'>
def qsort(collection)
  collection = [collection]
  Enumerator.new do |yielder|
    loop do
      smallers, *rest = collection
      pivot, *xs = smallers
      if pivot
        smaller = ->(x) { x < pivot }
        collection = [xs.select(&smaller),
                      pivot,
                      xs.reject(&smaller),
                      *rest]
      else
        sorted, *collection = *rest
        raise StopIteration unless sorted
        yielder.yield(sorted)
      end
    end
  end
end
</code></pre>

Let's walk through using this code to generate our first result.  First we
enter the method and wrap our array in another array, to emulate the data
structure we'll use when we start separating the lists recursively. We use
destructuring assignment to split that list into `smallers` and `rest`, which
since this our first loop, ends up being `[3, 4, 5, 3, 1, 2, 5, 4, 1]` and
`[]`.

<pre><code class='brush: ruby'>
def qsort(collection)
  collection = [collection]
  # [[3, 4, 5, 3, 1, 2, 5, 4, 1]]
  Enumerator.new do |yielder|
    loop do
      smallers, *rest = collection
      # [3, 4, 5, 3, 1, 2, 5, 4, 1], []
      pivot, *xs = smallers
      # 3, [4, 5, 3, 1, 2, 5, 4, 1]
      if pivot
        smaller = ->(x) { x < pivot }
        collection = [xs.select(&smaller),
                      pivot,
                      xs.reject(&smaller),
                      *rest]
        # [[1, 2, 1], 3, [4, 5, 3, 5, 4]]
        # return to beginning of the loop
      else
        # ...
      end
    end
  end
end
</code></pre>

We then split `smallers` into our pivot and the rest of the `smallers`, which
for lack of a better name I just borrowed `xs` from the Clojure example. We
get a `pivot` of 3, and an `xs` of `[4, 5, 3, 1, 2, 5, 4, 1]`. We enter the `if`
clause, since we have a pivot, and re-assign the `collection` variable to `[[1,
2, 1], 3, [4, 5, 3, 5, 4]]`.

Since there's nothing else to do, the loop continues again, emulating the
recursion in the Clojure example. We again divide up the collection, leaving us
with:

<pre><code class='brush: ruby'>
loop do
  smallers, *rest = collection
  # [1, 2, 1], [3, [4, 5, 3, 5, 4]]
  pivot, *xs = smallers
  # 1, [2, 1]
  if pivot
    smaller = ->(x) { x < pivot }
    collection = [xs.select(&smaller),
                  pivot,
                  xs.reject(&smaller),
                  *rest]
    # [[], 1, [2, 1], 3, [4, 5, 3, 5, 4]]
  else
    # ...
  end
end
</code></pre>

We lucked out and got a 1 as our pivot already, leaving our `smallers` list
empty on the next recursion:

<pre><code class='brush: ruby'>
loop do
  smallers, *rest = collection
  # [], [1, [2, 1], 3, [4, 5, 3, 5, 4]]
  pivot, *xs = smallers
  # nil, []
  if pivot
    # ...
  else
    sorted, *collection = *rest
    # 1, [[2, 1], 3, [4, 5, 3, 5, 4]]
    raise StopIteration unless sorted
    yielder.yield(sorted)
  end
end
</code></pre>

So we hop down to the else clause and yield the 1, and leave the rest of the
work until the method gets called upon to continue. By pushing the larger
numbers to the back of the list, we avoid sorting them, which in the best case
scenario might mean we never have to sort them at all. Some quick benchmarks
show what should be pretty obvious, that compared to a non-lazy quicksort
implementation, this performs way better when you only need a subset of the
list.

While I was experimenting with benchmarking, I discovered a possible
optimization where we could be EVEN LAZIER! Since in this algorithm we are
optimizing for sorting the smaller sets first, if we use Ruby 2.0's lazy
enumerable's on the larger sets, we can avoid doing the work of the `reject`
until we absolutely need to. Just by swapping

<pre><code class='brush: ruby'>
collection = [xs.select(&smaller),
              pivot,
              xs.reject(&smaller),
              *rest]
</code></pre>

for

<pre><code class='brush: ruby'>
collection = [xs.select(&smaller),
              pivot,
              xs.lazy.reject(&smaller),
              *rest]
</code></pre>

I was able to see the time for grabbing the first 20 results of an array of
100,000 random integers drop from 65ms to 28ms. Before you get hasty though,
and suggest we change the `select` to also be lazy, remember that that
collection gets immediately unpacked upon the next cycle of our loop, so adding
the overhead of laziness wouldn't buy us much here. Sure enough, when
benchmarked, using laziness on both the `reject` and the `select` increases the
time of the operation from 28ms to 69ms.

So what's the takeaway? Getting a better understanding of Enumerators is
definitely worthwhile, as they are a powerful feature of Ruby that can be a bit
confusing at first. I've been really impressed in Clojure with the simplicity
of lazy sequences, and so figuring out a way to reimplement them in Ruby is
pretty awesome. They offer an interesting alternative to recursion in a
language like Ruby that has no support for it, as well as many possibilities
for working through expensive calculations while only incurring the costs
needed at the given time. I still think the Ruby version is simpler to
understand, but that might just be my bias.
