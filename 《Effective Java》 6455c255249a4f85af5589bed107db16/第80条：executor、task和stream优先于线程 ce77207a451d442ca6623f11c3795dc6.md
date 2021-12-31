# 第80条：executor、task和stream优先于线程

在Java平台类库中新增了java.util.concurrent。这个包中包含了一个Executor Framework，它是一个很灵活的基于接口的任务执行工具。创建只需要一行代码：

```java
ExecutorService exec = Executors.newSingleThreadExecutor();
```

下面是为执行而提交一个runable的方法：

```java
exec.execute(runable);
```

下面是高数executor如何优雅地终止（如果没有这么做，虚拟机可能不会退出）：

```java
exec.shutdown();
```

你可以利用executor service完成更多的工作。例如，可以等待完成一项特殊的任务（就如第79条中的get方法一样），你可以等待一个任务集合中的任何任务或者所有任务完成（利用invokeAny或者invokeAll方法），可以等待executor service优雅地完成终止（利用awaitTermination方法），可以在任务完成时刻逐个地获取这些任务的结果（利用ExecutorCompletionService），可以调度在某个特殊的时间段定时运行或者阶段性地运行的任务（利用ScheduledThreadPoolExecutor），等等。

如果想让不止一个线程来处理来自这个队列的请求，只要调用一个不同的静态工厂，这个工厂创建了一种不同的executor service，称作线程池。你可以用固定或者可变数目的线程创建一个线程池。java.util.concurrent.Executor类包含了静态工厂，能为你提供所需要的大多数executor。然而，如果你想来点特别的，可以直接使用ThreadPoolExecutor类。这个列允许你控制线程池操作的几乎每个方面。

为特殊的应用程序原则executor service是很有技巧的。如果该编写的是小程序，或者是轻量负载的服务器，使用Executors.newCachedThreadPool通常是个不错的选择，因为它不需要配置，并且一般情况下能够正确的完成工作。但是对于大负载的服务器来说，缓存的线程就不是很好地选择了！在缓存的线程中，被提交的任务没有排成队列，而是直接交给线程执行。如果没有线程可用，就创建一个新的线程。如果服务器负载得太重，以致它所有的CPU都完全被占用了，当有更多地任务时，就会创建更多的线程，这样只会使情况变得更糟。因此，在大负载得产品服务中，最好使用Executors.newFixedThreadPool，它为你提供了一个包含固定线程数目的线程池，或者为了最大限度地控制它，就直接使用ThreadPoolExecutor类。

不仅应该尽量不要编写自己的工作队列，而且还应该尽量不直接使用线程。当直接使用线程时，Thread是既充当工作单元，又是执行机制。在Executor Framework中，工作单元和执行机制是分开的。现在关键的抽象是工作单元，称作任务（task）。任务有两种：Runnable及其近亲Callable（它与Runnable类似，但它会返回值，并且能够抛出任意的异常）。在执行任务的通用机制是executor service。如果你从任务的角度来看问题，并让一个executor service替你执行任务，在选择适当的执行策略方面就获得了极大地灵活性。从本质上讲，Executor Framework所做的工作是执行，Collections Framework所做的工作是聚合。

在Java 7中，Executor Framework得到了扩展，它可以支持fork-join任务了，这些任务是通过一种称作为fork-join池的特殊executor服务运行的。fork-join任务用ForkJoinTask实例表示，可以被分成更小的子任务，包含forkJoinPool的线程不仅要处理这些任务，还要从另一个线程中“偷”任务，以确保所有的线程保持忙碌，从而提高CPU使用率、提高吞吐量，并降低延迟。fork-join任务的编写和调优是很有技巧的。并发地stream是在fork join池上编写的，我们不费什么力气就能享受它们的性能优势，前提是假设它们正好适用于我们手边的任务。