---
layout: post
title: "Lock Striping"
permalink: /blog/2020/1/11/lock-striping
comments: true
github: "https://github.com/alimate/alimate.github.io/blob/master/_posts/2020-1-11-lock-striping.md"
excerpt: "A simple article to demonstrate the fact that how well a fine-grained synchronized concurrent data structure performs compared to its coarse-grained counterpart."
---
There are dozens of decent lock-free hashtable implementations. Usually those data structures, instead of using plain locks, are using *CAS* based operations to be *Lock Free*. With this narrative, it might sound I'm gonna use this post to build an argument for lock-free data structures. That's not the case, quite suprisingly. On the contrary, here we're going to talk about *Plain Old Locks*.

Now that everybody's on-board, Let's implement a concurrent version of `Map` with this simple contract:
{% highlight java %}
public interface ConcurrentMap<K, V> {

    boolean put(K key, V value);
    V get(K key);
    boolean remove(K key);
}
{% endhighlight %}
Since this is going to be a thread-safe data structure, we should *acquire* some locks and subsequently *release* them while putting, getting or removing elements:
{% highlight java %}
public abstract class AbstractConcurrentMap<K, V> implements ConcurrentMap<K, V> {

    // constructor, fields and other methods
    protected abstract void acquire(Entry<K, V> entry);
    protected abstract void release(Entry<K, V> entry);
}
{% endhighlight %}
Also, we should initialize the implementation with a pre-defined number of buckets:
{% highlight java %}
public abstract class AbstractConcurrentMap<K, V> implements ConcurrentMap<K, V> {

    protected static final int INITIAL_CAPACITY = 100;
    protected List<Entry<K, V>>[] table;
    protected int size = 0;

    public AbstractConcurrentMap() {
        table = (List<Entry<K, V>>[]) new List[INITIAL_CAPACITY];
        for (var i = 0; i < INITIAL_CAPACITY; i++) {
            table[i] = new ArrayList<>();
        }
    }

    // omitted
}
{% endhighlight %}

## One Size Fits All?
---
What happens when the number of stored elements are way more then the number of buckets? Even in the presence of the most versatile hash-functions, our hash table's performance would degrade signifcantly. In order to prevent this, we need to add more buckets when needed:
{% highlight java %}
protected abstract boolean shouldResize();
protected abstract void resize();
{% endhighlight %}
With the help of these abstract template methods, here's how we can put a new key-value pair into our map:
{% highlight java %}
@Override
public boolean put(K key, V value) {
    if (key == null) return false;

    var result = false;
    var entry = new Entry<>(key, value);
    acquire(entry);
    try {
        var bucketNumber = findBucket(entry);
        if (!table[bucketNumber].contains(entry)) {
            table[bucketNumber].add(entry);
            size++;
            result = true;
        }
    } finally {
        release(entry);
    }

    if (shouldResize()) resize();
    return result;
}

protected int findBucket(Entry<K, V> entry) {
    return abs(entry.hashCode()) % table.length;
}
{% endhighlight %}
In order to find a bucket for the new element, we are relying on the `hashcode` of the key. Also, we should calculate the `hashcode` modulo *the number of buckets* to prevent being out of bounds!

The remaining parts are pretty straightforward: We're acquiring a lock for the new entry and only add the entry if it does not exist already. At the end of the operation, if we came to this conclusion that we should add more
buckets, then we would do so to maintain the hash table's balance. The `get` and `remove` methods can also be implemented as easily.

Let's fill the blanks by implementing all those abstract methods!

## One Lock to Rule Them All
---
The simplest way to implement a thread-safe version of this data structure is to **use just one lock for all its operations**:
{% highlight java %}
public final class CoarseGrainedConcurrentMap<K, V> extends AbstractConcurrentMap<K, V> {

    private final ReentrantLock lock;

    public CoarseGrainedConcurrentMap() {
        super();
        lock = new ReentrantLock();
    }

    @Override
    protected void acquire(Entry<K, V> entry) {
        lock.lock();
    }

    @Override
    protected void release(Entry<K, V> entry) {
        lock.unlock();
    }

    // omitted
}
{% endhighlight %}
The current *acquire* and *release* implementations are completely independent from the map entry. As promised, we're using just one `ReentrantLock` for synchronization. This approach is known as **Coarse-Grained Synchronization**, since we're using just one lock to enforce exclusive access across the whole hashtable.

Also, we should resize the hash table to maintain its constant access and modification time. In order to do that, we can incorporate different heuristics. For example, when the average bucket size exceeds a specific number:
{% highlight java %}
@Override
protected boolean shouldResize() {
    return size / table.length >= 5;
} 
{% endhighlight %}
In order to add more buckets, we should copy the current hash table and create a new table with, say, twice the size. Then add all entries from the old table to the new one:
{% highlight java %}
@Override
protected void resize() {
    lock.lock();
    try {
        var oldTable = table;
        var doubledCapacity = oldTable.length * 2;

        table = (List<Entry<K, V>>[]) new List[doubledCapacity];
        for (var i = 0; i < doubledCapacity; i++) {
            table[i] = new ArrayList<>();
        }

        for (var entries : oldTable) {
            for (var entry : entries) {
                var newBucketNumber = abs(entry.hashCode()) % doubledCapacity;
                table[newBucketNumber].add(entry);
            }
        }
    } finally {
        lock.unlock();
    }
}
{% endhighlight %}
Please note that we should acquire and release the same lock for resize operation, too.

## Sequential Access
---
Suppose there are three concurrent requests for putting something into first bucket, getting something from the third bucket and removing something from the sixth bucket:
<p style="text-align:center">
  <img src="/images/hashtable-reqs.png" alt="Concurrent Requests">
</p>
Ideally we expect from a highly scalable concurrent data structure to serve such requests with as little coordination as possible. However, this is what happens in the read world:
<p style="text-align:center">
  <img src="/images/seq-access.png" alt="Sequential Access">
</p>
As we're using only one lock for synchronization, concurrent requests will be blocked behind the first one that acquired the lock. When the first one releases the lock, the second one acquires it and after sometime, releases it. 

This phenomenon is known as **Sequential Access** (Something like *Head of Line Blocking*) and we should mitigate this effect in concurrent environments.

## The United Stripes of Locks
---
One way to improve our current coarse-grained implementation is to **use a collection of locks instead of just one lock**. This way each lock would be responsible for synchronizing one or more buckets. This is more of a **Finer-Grained Synchronization** compared to our current approach:
<p style="text-align:center">
  <img src="/images/lock-striping.png" alt="Lock Striping">
</p>
Here we're dividing the hashtable into a few parts and synchronize each part independently. This way we reduce the contention for each lock and consequently improving the throughput. This idea also is called **Lock Striping** because we synchronize different parts (i.e. stripes) independently:
{% highlight java %}
public final class LockStripedConcurrentMap<K, V> extends AbstractConcurrentMap<K, V> {

    private final ReentrantLock[] locks;

    public LockStripedConcurrentMap() {
        super();
        locks = new ReentrantLock[INITIAL_CAPACITY];
        for (int i = 0; i < INITIAL_CAPACITY; i++) {
            locks[i] = new ReentrantLock();
        }
    }

    // omitted
}
{% endhighlight %}
We need a method to find an appropriate lock for each entry (i.e. bucket):
{% highlight java %}
private ReentrantLock lockFor(Entry<K, V> entry) {
    return locks[abs(entry.hashCode()) % locks.length];
}
{% endhighlight %}
Then we can acquire and release a lock for an operation on a particular entry as following:
{% highlight java %}
@Override
protected void acquire(Entry<K, V> entry) {
    lockFor(entry).lock();
}

@Override
protected void release(Entry<K, V> entry) {
    lockFor(entry).unlock();
}
{% endhighlight %}
The resize procedure is almost same as before. The only difference is that we should acquire all locks before the resize and release all of them after the resize.

We expect the fine-grained appraoch to outperform the coarse-grained one, let's measure it!

## Moment of Truth
---
In order to bechmark these two approaches, we're going to use the following workload on both implementations:
{% highlight java %}
final int NUMBER_OF_ITERATIONS = 100;
final List<String> KEYS = IntStream.rangeClosed(1, 100)
                                   .mapToObj(i -> "key" + i).collect(toList());

void workload(ConcurrentMap<String, String> map) {
    var requests = new CompletableFuture<?>[NUMBER_OF_ITERATIONS * 3];
    for (var i = 0; i < NUMBER_OF_ITERATIONS; i++) {
        requests[3 * i] = CompletableFuture.supplyAsync(
            () -> map.put(randomKey(), "value")
        );
        requests[3 * i + 1] = CompletableFuture.supplyAsync(
            () -> map.get(randomKey())
        );
        requests[3 * i + 2] = CompletableFuture.supplyAsync(
            () -> map.remove(randomKey())
        );
    }

    CompletableFuture.allOf(requests).join();
}

String randomKey() {
    return KEYS.get(ThreadLocalRandom.current().nextInt(100));
}
{% endhighlight %}
Each time we will put a random key, get a random key and remove a random key. Each workload would run these three opertions 100 times. Adding the JMH to the mix, each workload would be executed thousands of times:
{% highlight java %}
@Benchmark
@BenchmarkMode(Throughput)
public void forCoarseGrainedLocking() {
    workload(new CoarseGrainedConcurrentMap<>());
}

@Benchmark
@BenchmarkMode(Throughput)
public void forStripedLocking() {
    workload(new LockStripedConcurrentMap<>());
}
{% endhighlight %}
The coarse-grained implmentation can handle 8547 operations per second:
{% highlight text %}
8547.430 ±(99.9%) 58.803 ops/s [Average]
(min, avg, max) = (8350.369, 8547.430, 8676.476), stdev = 104.523
CI (99.9%): [8488.627, 8606.234] (assumes normal distribution)
{% endhighlight %}
On the other hand, the fine-grained implmentation can handle up to 13855 operations per second on average.
{% highlight text %}
13855.946 ±(99.9%) 119.109 ops/s [Average]
(min, avg, max) = (13524.367, 13855.946, 14319.174), stdev = 211.716
CI (99.9%): [13736.837, 13975.055] (assumes normal distribution)
{% endhighlight %}

## Recursive Split-Ordered Lists
---
In their [paper](https://ldhulipala.github.io/readings/split_ordered_lists.pdf), Ori Shalev and Nir Shavit proposed a way to implement a lock-free hashtable. They used a way of ordering elements in a linked list so that they can be repeatedly “split” using a single compare-and-swap operation. Anyway, they've used a pretty different approach to implement this concurrent data structure. It's highly recommended to checkout and read that paper.

Also, the source codes of this article is available on [Github](https://github.com/alimate/lock-striping).