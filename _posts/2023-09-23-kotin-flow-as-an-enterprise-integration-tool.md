---
layout: post
title: Kotin Flow as an Enterprise Integration tool
categories: [kotlin, async, nonblocking, reactive streams, reactive, flow, kotlin flow, coroutines, enterprise, integrations]
---

Kotlin Flow is a powerful reactive stream processing tool. Although not strictly based on the Reactive Streams specification, it's fully interoperable with it, and since it leverages coroutines, it makes it an incredibly fun tool to use. However, it is still not widely used for enterprise integration. I'm here to explain the whys and talk about how you can overcome these limitations and use Kotlin Flow as an ally when you're building enterprise integrations.

## Why Kotlin Flow?

First of all, why would you use Kotlin Flow for integrations, you may be asking. The answer is quite simple: it provides an efficient and elegant way to handle asynchronous data streams, making it ideal for building reactive and event-driven systems.

One of the key advantages of Kotlin Flow is its out-of-the-box support for ***backpressure***. ***Backpressure*** here refers to the ability to manage and control the flow of data when the rate of data production exceeds the rate of data consumption. This processing rate management is incredibly handy in enterprise applications, either if you don't want to overload the consumer, or if you don't want to make it idle all the time. In simple terms, by supporting backpressure, Kotlin Flow ensures that data is processed at a rate manageable by the downstream consumers.

Based on that, you could also be wondering: "***actually, Kotlin Flow seems to be an amazing tool to build enterprise integrations then!***”, but there are some caveats.

## Lack of built-in operators

While Kotlin Flow does offer some range of operators for transforming and combining data, they're definitely not enough to build more complex pipelines. First, when it comes to concurrency, there's no built-in operator for it. This means that if you have to handle complex concurrency scenarios, you may need to implement additional synchronization mechanisms, which isn't something we usually like to do when you just want to get things done:

```kotlin
val queueFlow: Flow<Message> = // some flow that consumes data from a queue

// as long as the producer uses coroutines, the processing is fully backpressured, so you don't have to worry about overflows.

queueFlow
    .map { message -> message.body.asJson<Order>() }
    .map { order -> saveToDB(order) } // one at a time. =(
    .collect { order -> log.info { "Order is successfully saved with id ${order.id}" } }
```

You can make the processing concurrent by using `**flatMapMerge**`, but it's more verbose and you lose the original order of the elements.

In addition to concurrency, implementing operators for chunking messages, splitting elements, and processing until timeouts complete are tasks you'll likely need to handle yourself.

## Lack of enterprise connectors

Kotlin Flow lacks not only some crucial operators for enterprise integrations but also the enterprise integrations themselves. Here, I am talking about message brokers, databases, external APIs, or any kind of I/O as a matter of fact. While Kotlin Flow does integrate easily with other Kotlin and Java libraries, you will need to implement the desired connectors to work with Kotlin Flow. The good news is that, since Kotlin Flow seamlessly interoperates with Reactive Streams, it's straightforward to use other connectors that also interoperate with Reactive Streams. However, this additional development can be challenging when you simply want to get things done.

Despite these limitations, I'm here to say that Kotlin Flow still offers a powerful solution for enterprise integrations, and the reason is that: you ***can*** implement all of those missing features, which means all the things that make Kotlin Flow flexible enough that still provide the very same capabilities that make it efficient for data stream processing are already in the library.

Shall we dive into some coding?

## Creating your first connector

First of all, let's create a simple connector. Let's not make something very complex, just something straightforward that simply works for most cases: a flow that actively polls for new data from a message broker.

The skeleton of the implementation is something like the code above:

```kotlin
fun messageQueueFlow(queueName: String): Flow<Message> =
    flow {
        while (true) {
            pollForMessages(queueName)
                .forEach { message -> emit(message) }
        }
    }
```

The `while(true) { ... }` construct may seem unusual if you're not accustomed to coroutines, but it makes total sense once you understand that your code is *suspendible*.

The `pollForMessages` implementation will depend on the message broker you're consuming the messages from. In this case, let's use Amazon SQS as the example:

```kotlin
val sqs: SqsAsyncClient = // create a SqsAsyncClient instance

// This map is used to cache the SQS queue urls.
// It's used to avoid having to always consult the queueUrl before each polling.
// It's a ConcurrentHashMap because you may have multiple consumers, so it has to be thread-safe
val queues = ConcurrentHashMap<String, String>()

suspend fun pollForMessages(queueName: String): List<Message> {
    val queueUrl = queues[queueName] ?: run {
        val url = sqs.getQueueUrl { it.name(queueName) }.await().queueUrl()
        queues[queueName] = url
        url
    }

    return sqs.receiveMessage { it.queueUrl(queueUrl) }.await().messages()
}
```

The implementation is now complete. However, be aware that we have not accounted for deleting the messages.

To start consuming messages from a specific queue, all you have to do is:

```kotlin
val queueFlow: Flow<Message> = messageQueueFlow("hello-world-queue")

queueFlow.collect { message -> log.info { it.body() } }
```

And *voilà*. You built your first connector using just a few lines of code. The polling isn't concurrent, but I challenge you to go and test it first. You'll be surprised at how fast it works, and you probably won't need concurrency for this specific step.

## Basic concurrency capabilities

The next step now is to build a manner to process the messages concurrently. As I said earlier, the data processing is sequential, including the `collect` step. If the same `collect` step takes some time for each message to be processed, you may think Kotlin Flow isn't quite the right tool for the job. Let's build a `map` that is capable of processing multiple elements concurrently, shall we?

Kotlin offers a Semaphore implementation, which is a tool that fits perfectly to help us address this challenge:

```kotlin
fun <T, R> Flow<T>.mapAsync(
    concurrencyLevel: Int,
    mapper: suspend (T) -> R
): Flow<R> =
    flow {
        val semaphore = Semaphore(concurrencyLevel)

        coroutineScope {
            map { element ->
                semaphore.acquire()
                async { mapper(element) }
            }
            .buffer(concurrencyLevel)
            .collect { promise ->
                emit(promise.await())
                semaphore.release()
            }
        }
    }
```

Now, just like `map`, you can also use `mapAsync` to process your messages:

```kotlin
val queueFlow: Flow<Message> = messageQueueFlow("hello-world-queue")

queueFlow
    .mapAsync(10) { message -> // do some heavy processing }
    .collect(...)
```

You can implement this same capability to `collect` itself without much code:

```kotlin
suspend fun <T> Flow<T>.collectAsync(
    concurrencyLevel,
    collector: suspend (T) -> Unit
) = mapAsync(concurrencyLevel, collector).collect()
```

The same can be done for `onEach`, `flatMap`, and so on.

## Integrating with other connectors

The options here are endless. If you're reading this far, you may have realized that Kotlin Flow can be a reactive, non-blocking, backpressure enabled, and type-safe alternative to Apache Camel. If you like the idea but are not eager to implement each missing operator and connector by yourself, you'll appreciate what I have to say next.

## What now?

I hope I have been able to explain why Kotlin Flow is advantageous and how it can assist you in building enterprise integrations. Implementing these missing capabilities can be tedious, and that's why I built [River](https://github.com/River-Kt/river/tree/main).

River is a collection of extension operators and connectors that makes enterprise integrations with Kotlin Flow not only easier, but really fun to do. You can take a look at the [website and see all of the available connectors](https://www.river-kt.com/), as well as checking out its [GitHub page](https://github.com/River-Kt/river), you'll notice how River serves as a versatile tool.

See you!
