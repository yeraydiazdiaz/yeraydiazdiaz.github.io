---
layout: post
title:  "AsyncIO for the Working Python Developer"
date:   2016-02-20 17:02:11 +0100
categories: blog
tags: python asyncio python3
image: asyncio-for-the-working-python-developer.jpg
---

I remember distinctly the moment where I thought, “Wow, that's slow, I bet if could parallelise these calls it would just fly!” and then, about three days later, I looked at my code and just didn't recognize it, it was an unreadable mash up of calls to threading and process library functions.

Then I found [asyncio](https://docs.python.org/dev/library/asyncio.html), and everything changed.

<!--more-->

If you don't know, *asyncio* is the new concurrency module introduced in Python 3.4. It's designed to use coroutines and futures to simplify asynchronous code and make it almost as readable as synchronous code simply because there are *no callbacks*.

I also remember that while on that quest for parallelisation a number of options were available, but one stood out. It was quick, easy to introduce and well thought of: the excellent [gevent](http://www.gevent.org/) library. I arrived at it by reading this lovely hands-on tutorial: [gevent for the Working Python Developer](http://sdiehl.github.io/gevent-tutorial/), written by an awesome community of users, a great introduction not only to gevent but to concurrency in general, and you most definitely should check it out.

I like the tutorial so much that I decided it would be a good template to follow when introducing *asyncio*.

Quick disclaimer, this is not a *gevent vs. asyncio* article, Nathan Road wrote a [great piece](http://www.getoffmalawn.com/blog/playing-with-asyncio) on what's similar and dissimilar between the two if you're interested.

**Update Feb 2017**: following some feedback I decided to use 3.5 async/await syntax, I've updated the examples accordingly. If you're interested the original 3.4 syntax examples are available in the [Github repo for this tutorial](https://github.com/yeraydiazdiaz/asyncio-ftwpd).

I know you're excited but before we dive in I'd like to quickly go over some concepts that may not be familiar at first.

## Threads, loops, coroutines and futures

Threads are a common tool and most developers have heard of and used before. However *asyncio* uses quite different constructs: *event loops*, *coroutines* and *futures*.

- An [*event loop*](https://docs.python.org/dev/library/asyncio-eventloop.html) essentially manages and distributes the execution of different tasks. It registers them and handles distributing the flow of control between them.
- [*Coroutines*](https://docs.python.org/dev/library/asyncio-task.html#coroutines) are special functions that work similarly Python generators that on **await** they release the flow of control back to the event loop. A coroutine needs to be scheduled to run using the event loop, to do this we create a *Task*, which is a type of *Future*.
- [*Futures*](https://docs.python.org/dev/library/asyncio-task.html#future) are objects that represent the result of a task that may or may not have been executed. This result may be an exception.

Got it? Pretty simple, right? let's dive right in!

## Synchronous & Asynchronous Execution

In [Concurrency is not parallelism, it's better](https://vimeo.com/49718712) Rob Pike makes a point that really made things click in my head. Breaking down tasks into concurrent subtasks only allows parallelism, it's the scheduling of these subtasks that creates it.

*Asyncio* does exactly that, you can structure your code so subtasks are defined as coroutines and allows you to schedule them as you please, including simultaneously. Coroutines contain yield points where we define possible points where a context switch can happen if other tasks are pending, but will not if no other task is pending.

A context switch in *asyncio* represents the event loop yielding the flow of control from one coroutine to the next. Let's have a look at a very basic example:

{% gist 5e5cf1354e83544666eafc7edb83e69e %}

```
$ python3 1-sync-async-execution-asyncio-await.py
Running in foo
Explicit context to bar
Explicit context switch to foo again
Implicit context switch back to bar
```

- First we declare a couple of simple coroutines that pretend to do non-blocking work using the `sleep` function in *asyncio*
- Coroutines can only be called from other coroutines or be wrapped in a task and then scheduled, we use `create_task`
- Once we have the two tasks we combine them into one that waits for both of them to complete using `wait`
- And finally we schedule the wait task to run using our event loop's `run_until_complete`

By using `await` on another coroutine we declare that the coroutine may give the control back to the event loop, in this case `sleep`. The coroutine will yield and the event loop will switch contexts to the next task scheduled for execution: *bar*. Similarly the *bar* coroutine uses `await sleep` which allows the event loop to pass control back to *foo* at the point where it yielded, just as normal Python generators.

Let's now simulate two blocking tasks, *gr1* and *gr2*, say they're two requests to external services. While those are executing a third task can be doing work asynchronously, like in the following example:

{% gist ae49c999ef6082d84c17c9837eedf593 %}

```
$ python3 1b-cooperatively-scheduled-asyncio-await.py
gr1 started work: at 0.0 seconds
gr2 started work: at 0.0 seconds
Lets do some stuff while the coroutines are blocked, at 0.0 seconds
Done!
gr1 ended work: at 2.0 seconds
gr2 Ended work: at 2.0 seconds
```

Notice how the I/O loop manages and schedules the execution allowing your single threaded code to operate concurrently. While the two blocking tasks are blocked a third one can take control of the flow.

## Order of execution

In the synchronous world we're used to thinking linearly. If we were to have a series of tasks that take different amounts of time they will be executed in the order that they were called upon.

However, when using concurrency we need to be aware that the tasks finish in different order than they were scheduled.

{% gist dca724d79e2476ebeaf747b989b6cac0 %}

```
$ python3 1c-determinism-sync-async-asyncio-await.py
Synchronous:
Task 1 done
Task 2 done
Task 3 done
Task 4 done
Task 5 done
Task 6 done
Task 7 done
Task 8 done
Task 9 done
Asynchronous:
Task 2 done
Task 5 done
Task 6 done
Task 8 done
Task 9 done
Task 1 done
Task 4 done
Task 3 done
Task 7 done
```

Your output will, of course, vary since each task will sleep for a random amount of time, but notice how the resulting order is completely different, even though we built the array of tasks in the same order using **range**.

Also notice how we had to create a coroutine version of our fairly simple task. It's important to understand that *asyncio* does not magically make things non-blocking. At the time of writing *asyncio* stands alone in the standard library, the rest of modules provide only blocking functionality. You can use the [concurrent.futures](https://docs.python.org/3/library/concurrent.futures.html#module-concurrent.futures) module to wrap a blocking task in a thread or a process and return a Future *asyncio* can use. This same example using threads is available in the [Github repo](https://github.com/yeraydiazdiaz/asyncio-ftwpd).

This is probably the main drawback right now when using *asyncio*, however there are plenty of libraries for different tasks and services.

A very common blocking task is, of course, fetching data from an HTTP service. I'm using the excellent [aiohttp](http://tp.readthedocs.org/) library for non-blocking HTTP requests retrieving data from Github's public event API and simply take the Date response header.

{% gist f8a1d19d5324402825f745cb4d31b797 %}

```
$ python3 1d-async-fetch-from-server-asyncio-await.py
Synchronous:
Fetch sync process 1 started
Process 1: Wed, 17 Feb 2016 13:10:11 GMT, took: 0.54 seconds
Fetch sync process 2 started
Process 2: Wed, 17 Feb 2016 13:10:11 GMT, took: 0.50 seconds
Fetch sync process 3 started
Process 3: Wed, 17 Feb 2016 13:10:12 GMT, took: 0.48 seconds
Process took: 1.54 seconds
Asynchronous:
Fetch async process 1 started
Fetch async process 2 started
Fetch async process 3 started
Process 3: Wed, 17 Feb 2016 13:10:12 GMT, took: 0.50 seconds
Process 2: Wed, 17 Feb 2016 13:10:12 GMT, took: 0.52 seconds
Process 1: Wed, 17 Feb 2016 13:10:12 GMT, took: 0.54 seconds
Process took: 0.54 seconds
```

First off, note the difference in timing, by using asynchronous calls we're making *at the same time* all the requests to the service. As discussed each request yields the control flow to the next and returns when it's completed. The result is that requesting and retrieving the result of all requests takes only as long as the slowest request! See how the timing log is 0.54 seconds for the slowest request which is the same for the total time elapsed by processing all the requests. Pretty cool, huh?

Secondly look at how similar the code is to the synchronous version! It's essentially the same! The main differences are due to library implementation for performing the GET request and creating the tasks and waiting for them to finishing.

## Creating concurrency

So far we've been using a single method of creating and retrieving results from coroutines, creating a set of tasks and waiting for all of them to finish.

But coroutines can be scheduled to run or retrieve their results in different ways. Imagine a scenario where we need to process the results of the HTTP GET requests as soon as they arrive, the process is actually quite similar than in our previous example:

{% gist 4d7444e117a6e516634d09a73df1ccdb %}

```
$ python3 2a-async-fetch-from-server-as-completed-asyncio-await.py
Fetch async process 1 started, sleeping for 4 seconds
Fetch async process 3 started, sleeping for 5 seconds
Fetch async process 2 started, sleeping for 3 seconds
>> Process 2: Wed, 17 Feb 2016 13:55:19 GMT, took: 3.53 seconds
>>>> Process 1: Wed, 17 Feb 2016 13:55:20 GMT, took: 4.49 seconds
>>>>>> Process 3: Wed, 17 Feb 2016 13:55:21 GMT, took: 5.48 seconds
Process took: 5.48 seconds
```

Note the padding and the timing of each result call, they are scheduled at the same time, the results arrive out of order and we process them as soon as they do.

The code in this case is only slightly different, we're gathering the coroutines into a list, each of them ready to be scheduled and executed. The [`as_completed`](https://docs.python.org/dev/library/asyncio-task.html#asyncio.as_completed) function returns an iterator that will yield a completed future as they come in. Now don't tell me that's not cool. By the way, `as_completed` and `wait` are both functions originally from [concurrent.futures](https://docs.python.org/dev/library/concurrent.futures.html#module-functions).

Let's get to another example, imagine you're trying to get your IP address. There are similar services you can use to retrieve it but you're not sure if they will be accessible at runtime. You don't want to check each one sequentially, ew. You would send concurrent requests to each service and pick the first one that responds, right? Right!

Well, turns out our old friend [`wait`](https://docs.python.org/dev/library/asyncio-task.html#asyncio.wait) has a parameter to do just that: `return_when`. Up until now we were ignoring what wait returns since we were just parallelising tasks. But now we want to retrieve the results from the coroutine, so we can use the two sets of futures, *done* and *pending*.

{% gist 37bde9c9ac39c4780a9c456f092820f7 %}

```
$ python3 2c-fetch-first-ip-address-response-await.py
Fetching IP from ip-api
Fetching IP from ipify
ip-api finished with result: 82.34.76.170, took: 0.09 seconds
Unclosed client session
client_session: <aiohttp.client.ClientSession object at 0x10f95c6d8>
Task was destroyed but it is pending!
task: <Task pending coro=<fetch_ip() running at 2c-fetch-first-ip-address-response.py:20> wait_for=<Future pending cb=[BaseSelectorEventLoop._sock_connect_done(10)(), Task._wakeup()]>>
```

Wait, what happened there? The first service responded just fine but what's with all those warnings?

Well, we scheduled two tasks but once the first one completed the closed the loop leaving the second one pending. *Asyncio* assumes that's a bug and prints out a warning. We really should clean up after ourselves and let the event loop know not to bother with the pending futures. How? Glad you asked.

## Future states

(As in states that a Future can be in, not states that are in the future… you know what I mean)

These are:

- Pending
- Running
- Done
- Cancelled

As simple as that. When a future is done it's result method will return the result of the future, if it's pending or cancelled it raises `InvalidStateError`, if it's cancelled it will raise `CancelledError`, and finally if the coroutine raised an exception it will be raised again, which is the same behaviour as calling `exception`. [But don't take my word for it](https://docs.python.org/dev/library/asyncio-task.html#future).

You can also call `done`, `cancelled` or `running` on a Future to get a boolean if the Future is in that state, note that "done" simply means `result` will return or raise and exception. You can specifically cancel a Future by calling the `cancel` method (oddly enough), which sounds like what we need to fix the warning in the previous example:

{% gist 4b88bf520d5adcdc42e57a22deddbb26 %}

```
$ python3 2c-fetch-first-ip-address-response-no-warning-await.py
Fetching IP from ipify
Fetching IP from ip-api
ip-api finished with result: 82.34.76.170, took: 0.08 seconds
```

Nice and tidy output, gotta love it.

Futures also allow attaching callbacks when they get to the done state in case you want to add additional logic. You can even manually set the result or the exception of a Future, typically for unit testing purposes.

## Exception handling

*Asyncio* is all about making concurrent code manageable and readable, and that becomes really obvious in the handling of exceptions. Let's go back to an example to illustrate this.

Imagine we want to ensure all our IP services return the same result, but one of our services is offline and not resolving. We can simply use `try...except`, as usual:

{% gist 69db8556011c12a600a1a59a975f897f %}

```
$ python3 3a-fetch-ip-addresses-fail-await.py
Fetching IP from ip-api
Fetching IP from borken
Fetching IP from ipify
ip-api finished with result: 85.133.69.250, took: 0.75 seconds
ipify finished with result: 85.133.69.250, took: 1.37 seconds
borken is unresponsive
```

We can also handle the exceptions as we process the results of the futures, in case an unexpected exception occurred:

{% gist 2f4eba8177614c80e3641c836f3c9252 %}

```
$ python3 3b-fetch-ip-addresses-future-exceptions-await.py
Fetching IP from ipify
Fetching IP from borken
Fetching IP from ip-api
ipify finished with result: 85.133.69.250, took: 0.91 seconds
borken is unresponsive
Unexpected error: Traceback (most recent call last):
 File “3b-fetch-ip-addresses-future-exceptions.py”, line 39, in asynchronous
 print(future.result())
 File “3b-fetch-ip-addresses-future-exceptions.py”, line 26, in fetch_ip
 ip = json_response[service.ip_attr]
KeyError: 'this-is-not-an-attr'
```

Didn't see that one coming…

In the same way that scheduling a task and not waiting for it to finish is considered a bug, scheduling a task and not retrieving the possible exceptions raised will also throw a warning:

{% gist b3b3ad6102fab0bcd77bb21b03b22a1f %}

```
$ python3 3c-fetch-ip-addresses-ignore-exceptions-await.py
Fetching IP from ipify
Fetching IP from borken
Fetching IP from ip-api
borken is unresponsive
ipify finished with result: 85.133.69.250, took: 0.78 seconds
Task exception was never retrieved
future: <Task finished coro=<fetch_ip() done, defined at 3c-fetch-ip-addresses-ignore-exceptions.py:15> exception=KeyError('this-is-not-an-attr',)>
Traceback (most recent call last):
 File “3c-fetch-ip-addresses-ignore-exceptions.py”, line 25, in fetch_ip
 ip = json_response[service.ip_attr]
KeyError: 'this-is-not-an-attr'
```

That looks remarkably like the output from our previous example, minus the tut-tut message from *asyncio*.

## Timeouts

What if we don't really care that much about our IP? Imagine it being a nice addition to a more complex response but we certainly don't want to keep the user waiting for it. Ideally we'd give our non-blocking calls a timeout, after which we just send our complex response without the IP attribute.

Again `wait` has just the attribute we need:

{% gist 77c487c257c638d0524694c0d1affff7 %}

Notice the `timeout` argument on `wait`, we're also adding a command line argument to test what happens if we do allow the requests some time. I also added a some random sleeping time to ensure things didn't move too fast.

```
$ python 4a-timeout-with-wait-kwarg-await.py
Using a 0.01 timeout
Fetching IP from ipify
Fetching IP from ip-api
{'message': 'Result from asynchronous.', 'ip': 'not available'}

$ python 4a-timeout-with-wait-kwarg-await.py -t 5
Using a 5.0 timeout
Fetching IP from ip-api
Fetching IP from ipify
ipify finished with result: 82.34.76.170, took: 1.24 seconds
{'ip': '82.34.76.170', 'message': 'Result from asynchronous.'}
```

## Conclusion

*Asyncio* has extended my already ample love for Python. To be absolutely honest I fell in love with marriage of coroutines and Python when I first discovered [Tornado](http://tornadoweb.org/) but *asyncio* has managed to unify the best of this and the rest of excellent concurrency libraries into a rock solid piece. So much so that a special effort was made to ensure these and other libraries can use the main IO loop, so if you're using *Tornado* or [*Twisted*](https://www.twistedmatrix.com/) you can make use of libraries intended for *asyncio*!

As I said before its main problem is the lack of standard library modules that implement non-blocking behaviour. You may find that a particular technology that has plenty of well established Python libraries to interact with will not have a non-blocking version, or the existing ones are young lived or experimental. However, [the number of libraries is only getting bigger](http://asyncio.org/#libraries).

Hopefully in this tutorial I communicated what a joy is to work with *asyncio*. I honestly think it's the piece that will finally make adaptation to Python 3 a reality, it really feels you're missing out if you're stuck with Python 2.7. One thing's for sure, Python's future has completely changed, pun intended.