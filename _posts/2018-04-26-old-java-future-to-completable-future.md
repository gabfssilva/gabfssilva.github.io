---
layout: post
title: Transforming Java's old Future into CompletableFuture
categories: [java, async, future]
---

Many asynchronous libraries in Java still do not support CompletableFuture, which often leaves us dealing with the traditional Java Future. The problem is that it lacks of callback methods, forcing us to block the execution to get a result, something that is far from ideal.

Let's talk about a straightforward method to convert the traditional Future into the more efficient CompletableFuture using plain Java, no additional libraries required.

As said previously, Java Future interface lacks any callback method, so the only thing left for us to do, if you want to know when it's done without blocking any threads, is via an event loop. This loop checks the `isDone` and `isCancelled` methods, completes it with a promise (a `CompletableFuture` in our case), and then stops the event loop once it's completed.

To optimize CPU usage, we'll confine this event loop to a single thread. Java provides an easy way to create an event loop using the `ScheduledExecutorService`:


```java
public class CompletablePromiseContext {
    private static final ScheduledExecutorService SERVICE = Executors.newSingleThreadScheduledExecutor();
    
    public static void schedule(Runnable r) {
        SERVICE.schedule(r, 1, TimeUnit.MILLISECONDS);
    }
}
```

Next, we'll create a class that will run the event loop for each future:

```java
public class CompletablePromise<V> extends CompletableFuture<V> {
    private Future<V> future;

    public CompletablePromise(Future<V> future) {
        this.future = future;
        CompletablePromiseContext.schedule(this::tryToComplete);
    }

    private void tryToComplete() {
        if (future.isDone()) {
            try {
                complete(future.get());
            } catch (InterruptedException e) {
                completeExceptionally(e);
            } catch (ExecutionException e) {
                completeExceptionally(e.getCause());
            }
            return;
        }

        if (future.isCancelled()) {
            cancel(true);
            return;
        }

        CompletablePromiseContext.schedule(this::tryToComplete);
    }
}
```

In the `CompletablePromise` class, we schedule the `tryToComplete` method when creating a `CompletablePromise`, which is a `CompletableFuture`. The `tryToComplete` method checks if the future is completed or cancelled. If an exception occurs, the `completeExceptionally` method is invoked. If it is not completed, it schedules the method again to continue the event loop.

Now, in order to convert an old Java Future into a CompletableFuture, you simply have to:

```java
public class Main {
    public static void main(String[] args) {
        final ExecutorService service = Executors.newSingleThreadExecutor();
        final Future<String> stringFuture = service.submit(() -> "success");
        final CompletableFuture<String> completableFuture = new CompletablePromise<>(stringFuture);

        completableFuture.whenComplete((result, failure) -> {
            System.out.println(result);
        });
    }
}
```


I hope you find this guide helpful. Happy coding!
