---
layout: post
title: "Patching ThreadPoolExecutor to handle Errors"
description: ""
category: java
tags: [java, quality]
---

In this post I'll describe an important patch that you _always_ want to use
when using a `ThreadPoolExecutor` (or any `ExecutorService`) in Java.

<!-- more -->

The patch intends to mitigate the unexpected death of threads, and the impact
that they should be allowed to have on your application.

The help illustrate illustrate this, here is an example project with a very
nasty thread eating memory in it:

{% highlight java %}
public class Example {
    private static final int MESSAGE_SIZE = 1024 * 1000;

    public static void main(String[] argv) throws Exception {
        final ExecutorService executor =
            new ThreadPoolExecutor(2, 2, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>());

        final TransferQueue<long[]> queue = new LinkedTransferQueue<>();

        executor.submit(new BadThread());

        executor.submit(() -> {
            while (true) {
                queue.transfer(new long[MESSAGE_SIZE]);
            }
        });

        while (true) {
            System.out.println("main: waiting for message...");
            queue.take();
            System.out.println("main: OK");
            Thread.sleep(500);
        }
    }

    /**
     * A bad thread eating up all available memory and holding on to it.
     */
    static class BadThread implements Callable<Void> {
        @Override
        public Void call() throws Exception {
            Thread.sleep(1000);

            System.out.println("BadThread: Start 'borrowing' memory...");

            final List<Long> list = new ArrayList<>();

            while (true) {
                try {
                    list.add(0L);
                } catch (final OutOfMemoryError error) {
                    System.out.println("BadThread: Hold on to OOM: " + error);
                    Thread.sleep(10000);
                }
            }
        }
    }
}
{% endhighlight %}

Compile and run this application with `-Xmx16m`.
You should see something like the following:

{% highlight bash %}
main: waiting for message...
main: OK
main: waiting for message...
main: OK
BadThread: Start 'borrowing' memory...
main: waiting for message...
main: OK
BadThread: Hold on to OOM: java.lang.OutOfMemoryError: Java heap space
main: waiting for message...
...
{% endhighlight %}

The application is stuck, we are no longer seeing any `main: OK` messages.
No stack traces, nothing.

The reason is that out coordinator thread allocates memory for its message,
this means that it could, and is the target of an `OutOfMemoryError` when the
allocation fails because `BadThread` has locked up all available memory and is
refusing to die.

This is when it gets interesting. `ThreadPoolExecutor` will happily catch and
swallow any exception being thrown in one of its tasks. As per the
documentation, it is explicitly left to the developer to handle this.

This leaves us with a coordinator at the end of the `Queue` to die, and `main`
is left to its own devices forever. :(.

# The afterExecute patch

This patch is derived from [this StackOverflow answer](http://stackoverflow.com/questions/2248131/handling-exceptions-from-java-executorservice-tasks) and can be applied to `ThreadPoolExecutor`.

{% highlight java %}
final ExecutorService executor = new ThreadPoolExecutor(2, 2, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>()) {
    protected void afterExecute(Runnable r, Throwable t) {
        super.afterExecute(r, t);

        if (t == null && r instanceof Future<?>) {
            try {
                Future<?> future = (Future<?>) r;

                if (future.isDone()) {
                    future.get();
                }
            } catch (CancellationException ce) {
                t = ce;
            } catch (ExecutionException ee) {
                t = ee.getCause();
            } catch (InterruptedException ie) {
                Thread.currentThread().interrupt(); // ignore/reset
            }
        }

        if (t != null) {
            if (t instanceof Error) {
                try {
                    System.err.println("Error in runnable: " + r);
                    t.printStackTrace(System.err);
                    System.err.println(
                        "This is an unrecoverable error, shutting down...");
                } finally {
                    System.exit(1);
                }
            }

            System.out.println(t);
        }
    }
};
{% endhighlight %}

This patch overrides the [`afterExecute`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadPoolExecutor.html#afterExecute-java.lang.Runnable-java.lang.Throwable-)
 method. A hook designed to allow for custom behavior after the completion of
tasks.

Run the project again, and you should see the following:

{% highlight bash %}
main: waiting for message...
main: OK
main: waiting for message...
main: OK
BadThread: Start 'borrowing' memory...
main: waiting for message...
main: OK
BadThread: Hold on to OOM: java.lang.OutOfMemoryError: Java heap space
Error in runnable: java.util.concurrent.FutureTask@5cf149bb
java.lang.OutOfMemoryError: Java heap space
    at com.spotify.heroic.ExecutorServicePatch.lambda$main$0(ExecutorServicePatch.java:63)
    at com.spotify.heroic.ExecutorServicePatch$$Lambda$1/495053715.call(Unknown Source)
    at java.util.concurrent.FutureTask.run(FutureTask.java:266)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
    at java.lang.Thread.run(Thread.java:745)
This is an unrecoverable error, shutting down...

Process finished with exit code 1
{% endhighlight %}

# Errors

I want to emphasise that `OutOfMemoryError` is generally not an error that you
can safely recover from. There are no guarantees that the thread responsible
for eating up your memory is the target for this error. Even if that is the
case, this thread might become important at a later stage in its life.
In my opinion, the most reasonable thing to do is to give up.

<blockquote>
  An Error is a subclass of Throwable that indicates serious problems that a
  reasonable application should not try to catch. Most such errors are abnormal
  conditions.
  <footer>
    &mdash;
    <cite><a href="https://docs.oracle.com/javase/8/docs/api/java/lang/Error.html">https://docs.oracle.com/javase/8/docs/api/java/lang/Error.html</a></cite>
  </footer>
</blockquote>

At this stage you might be tempted to attempt a clean shutdown of your
application on errors.
This might work. But we might also be in a state where a thread critical
towards the clean shutdown of your application is no longer alive.
There might not be any memory left to support a complex shutdown. Attempting it
could lead to your cleanup attempt crashing leading us back to where we
started.

If you want to cover manually created threads, you can make use of
[`Thread#setDefaultUncaughtExceptionHandler`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#setDefaultUncaughtExceptionHandler-java.lang.Thread.UncaughtExceptionHandler-).
Just remember, this still does not cover thread pools.

On a final note, if you are a library developer: Please don't hide your thread
pools from us.
