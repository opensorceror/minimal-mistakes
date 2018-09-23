---
title:  "Asynchronous programming with Scala Futures"
header:
  caption: "Photo credit: [**Official Scala logo**](https://www.scala-lang.org/)"
  teaser: assets/images/scala-spiral.png
excerpt: "Demystifying asynchronous programming with Scala"
categories:
  - general-programming

---

{% include toc title="Navigation" %}

Asynchronous programming can be a daunting subject when first encountered by programmers. This post aims to serve as an introductory tutorial for asynchronous programming using the  `Future` construct in Scala.

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">A programmer had a problem. He thought to himself, &quot;I know, I&#39;ll solve it with threads!&quot;. has Now problems. two he</p>&mdash; Davidlohr Bueso (@davidlohr) <a href="https://twitter.com/davidlohr/status/288786300067270656?ref_src=twsrc%5Etfw">January 8, 2013</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

For a more in-depth look at asynchronous programming with Scala, check out the [official docs][scala-futures-docs]. 
{: .notice--info}

## Why asynchronous?

Consider that we have to execute $$N$$ *independent* tasks, denoted by $$T_1, T_2, ..., T_n$$. One approach is to execute them sequentially, as is commonly seen in synchronous programming (Fig. 1):

{% include figure image_path="/assets/images/2018-09-22-asynchronous-programming-scala-future/tasks_sequential.svg" alt="Sequential execution" caption="Figure 1: Sequential execution" %}{: .align-left .svg-width-20}

Although this may be sufficient in many cases, it may not be sufficient if all $$N$$ tasks are required to be completed with a certain time constraint. Furthermore, one or more of the tasks in the collection may take much longer to complete than others, thus becoming a bottleneck. For example, assume that $$N = 10$$. Now, if $$T_2$$ takes 50 minutes to complete while the other nine tasks take $$< 1$$ minute, tasks $$T_3, T_4, ..., T_n$$ will unnecessarily wait for $$T_2$$ to complete, despite being independent from it.

A more efficient approach may be to launch all $$N$$ tasks in parallel (Fig. 2). The number of tasks that are effectively executed in parallel is termed the *parallelism level* or *degree of parallelism*. In practice, the parallelism level is limited by the number of threads and/or cores in the processor.

{% include figure image_path="/assets/images/2018-09-22-asynchronous-programming-scala-future/tasks_parallel.svg" alt="Parallel execution" caption="Figure 2: Parallel execution" %}{: .align-right .svg-width-70}

{% include figure image_path="/assets/images/2018-09-22-asynchronous-programming-scala-future/tasks_longrunning.svg" alt="Asynchronous execution of long running task" caption="Figure 3: Asynchronous execution of long running task" %}{: .align-right .svg-width-40}  

Another approach is to only launch certain tasks asynchronously, such as those that run much longer than others (Fig. 3). Coming back to our example where $$T_2$$ runs much longer than other tasks, we can launch $$T_2$$ asynchronously after $$T_1$$. By doing this, the execution will immediately move on to $$T_3$$ without waiting for $$T_2$$ to complete, while $$T_2$$ will continue to run on a separate *thread*. When $$T_2$$ completes, its result can be obtained using a *callback* (discussed shortly).

## Parallelism vs concurrency
Before proceeding, it is important to distinguish between *parallelism* and *concurrency*. Although they may look like synonyms in everyday usage, these terms are defined differently in computer science. 

For an interesting detailed discussion, refer to [Jenkov's excellent blog post][jenkov-concurrency-vs-parallelism]. Briefly, concurrency represents tasks that *seemingly* run in parallel, while parallelism represents tasks that actually run in parallel. When a processor runs tasks concurrently, they may all be making progress, but the processor is actually only running one or a subset of them at any instant in time. This is usually achieved by rapidly switching context from one task to another. On the other hand, when a processor is running tasks in parallel, it is actually running all of them simultaneously.

So where does *asynchronous programming* fit in then? Well, it may imply either concurrency or parallelism. If the processor supports running tasks in parallel, asynchronously launched tasks will run in parallel. If the processor can only run one task at a time, asynchronously launched tasks will run concurrently. 

## Asynchronous programming in Scala
Scala provides simple, elegant high level APIs to execute code asynchronously. This section briefly discusses *Futures* and related concepts.

### Using Futures
Simply put, the aptly named `Future` object represents any code that may run asynchronously and produce a result in the future.

Here's an example of a `Future`:

{% highlight scala %}

import scala.concurrent.ExecutionContext.Implicits.global

val sorted: Future[SortedSequence] = {
  abominablySlowSortingAlgorithm(abominablyLargeSequence) // abominably long lasting computation
}

{% endhighlight %}

Note the import. This is an *implicit* global static thread pool that is used to execute the `Future`. It is also possible to explicitly specify an *`ExecutionContext`* with a user-defined thread pool; for a detailed discussion please refer to the [official docs][scala-futures-docs]. 

The `sorted` value here is a `Future` that embodies the result of the long lasting sort operation. 

### Callbacks
Usually, we are interested in obtaining the results from an asynchronous computation. This can be achieved by using a *callback*. A callback is simply a function that is called when a `Future` is completed. Since higher-order functions are first class citizens in Scala, the definition of callbacks is easily achieved. 

For a `Future`, a callback is typically registered by supplying a function of type `Try[T] => U` to the `onComplete` method. If the `Future` is successful, the supplied callback is applied to the value of type `Success[T]`, or to a value of type `Failure[T]` otherwise. 

Coming back to our sorting example, we can now register the callback as follows:

{% highlight scala %}

import scala.concurrent.ExecutionContext.Implicits.global

val sorted: Future[SortedSequence] = {
  abominablySlowSortingAlgorithm(abominablyLargeSequence) // abominably long lasting computation
}

sorted onComplete {
  case Success(value) => println(s"Sorted sequence: $value")
  case Failure(e) => println(s"An abominable error has occured: ${e.getMessage}")
}

{% endhighlight %}

If the `Future` is successful, the sorted sequence will be printed. If, for any reason the `Future` fails, the exception message will be printed. 

### Scaling Futures

Suppose we need to launch 100 tasks in parallel, and then do something with the results of the 100 tasks. Creating one future per task and registering a callback for each would be extremely tedious, not to mention unscalable. One solution is to create a sequence of `Future[T]` objects and convert this `Seq[Future[T]]` object into a single `Future[Seq[T]]` object, so we can do something with the results collected from all futures.

An example will help illustrate the idea:

{% highlight scala linenos %}

import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._
import scala.concurrent.{Await, Future}

lazy val t1 = Future[String] {
  println("Task 1 running!")
  Thread.sleep(10000)
  println("Task 1 done!")
  "result from task 1"
}

lazy val t2 = Future[String] {
  println("Task 2 running...")
  throw new RuntimeException("Task 2 fails")
}

lazy val t3 = Future[String] {
  println("Task 3 running...")
  Thread.sleep(5000)
  println("Task 3 done!")
  "result from task 3"
}

val futures = Future.sequence(Seq[Future[String]](t1, t2, t3))

futures.onSuccess {
  case value => println(s"**Drumroll** And the results are...: $value")
}

Await.ready(futures, 20 seconds)

{% endhighlight %}

On line 24, we converting a `Seq[Future[String]]` object into a `Future[Seq[String]]` object. Note that internally, what this actually does is: 
1. Executes the Futures in the sequence asynchronously
2. Once all Futures are complete (whether successfully or otherwise), combines the result of all Futures into a `Future[Seq[String]]` object.

However, there is a problem with this code. We're defining three Futures, and one of them throws a `RuntimeException`. Can you guess what will happen here? 

Here's the output:

```scala
Task 1 running!
Task 3 running...
Task 2 running...
Task 3 done!
Task 1 done!
```

Oh wait, where are the results of the tasks that we're printing on line 27? 

### Composing Futures
The reason the results of the tasks were not printed to the console is as follows: If any of the Futures in a sequence of Futures fails, the `onSuccess` function does not get executed. And since we did not define an `onFailure` function, the exception was ignored! What now?

Scala's powerful functional personality comes to the rescue again! Scala allows composing Futures using *combinators*. Briefly, combinators are functions that operate on Futures and return new `Future` objects. Scala providers several types of combinators, including `flatMap`, `filter`, `foreach`, etc.

One particular combinator that will help resolve our sticky situation is `recover`. When applied on a `Future`, the `recover` combinator allows intercepting an exception and doing something with it, and potentially returning a default value instead. Using `recover`, we can rewrite our example as:

{% highlight scala linenos %}

import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.duration._
import scala.concurrent.{Await, Future}

lazy val t1 = Future[String] {
  ...
}

lazy val t2 = Future[String] {
  println("Task 2 running...")
  throw new RuntimeException("Task 2 fails")
} recover {
    case e: Exception => e.getMessage
}

lazy val t3 = Future[String] {
  ...
}

val futures = Future.sequence(Seq[Future[String]](t1, t2, t3))

futures.onSuccess {
  case value => println(s"**Drumroll** And the results are...: $value")
}

Await.ready(futures, 20 seconds)

{% endhighlight %}

Output:

```scala
Task 1 running!
Task 2 running...
Task 3 running...
Task 3 done!
Task 1 done!
**Drumroll** And the results are...: List(result from task 1, Task 2 fails, result from task 3)
```

That works as expected! Now you know about asynchronous programming in Scala! I'd love to hear your thoughts about this post, please leave a comment below!

{% include figure image_path="/assets/images/2018-09-22-asynchronous-programming-scala-future/thats-all-folks.png" alt="That's all folks!" %}{: .align-center}

[scala-futures-docs]: https://docs.scala-lang.org/overviews/core/futures.html
[jenkov-concurrency-vs-parallelism]: http://tutorials.jenkov.com/java-concurrency/concurrency-vs-parallelism.html