---
layout: post
title:  "AsyncIO for the Working Python Developer"
date:   2015-12-06 17:02:11 +0100
last_modified: 2018-06-28 12:36:48 +0000
categories: python asyncio
tags: python asyncio python3
image: asyncio-for-the-working-python-developer.jpg
short_description: A hands on tutorial on Python's asyncio explaining concurrency with simple examples working up to making concurrent HTTP calls.
keywords: "python, tutorial, asyncio, async, example, simple, http, concurrency, task, future"
applause: true
canonical_url: https://medium.com/@yeraydiazdiaz/asyncio-for-the-working-python-developer-5c468e6e2e8e
---

<div markdown="1" class="sticky">
* TOC
{:toc}
</div>

<div markdown="1" id="text">
I remember distinctly the moment where I thought, “Wow, that's slow, I bet if could parallelise these calls it would just fly!” and then, about three days later, I looked at my code and just didn't recognize it, it was an unreadable mash up of calls to threading and process library functions.

Then I found [asyncio](https://docs.python.org/dev/library/asyncio.html), and everything changed.

<!--more-->

<div class="note">
<a href="{{ page.canonical_url }}" target="_blank"><em>Asyncio for the Working Python Developer</em> was first published on Medium</a>, if you'd like to comment or give feedback please do so there.
</div>

If you don’t know, *asyncio* is the new concurrency module introduced in Python 3.4. It’s designed to use coroutines and futures to simplify asynchronous code and make it almost as readable as synchronous code simply because there are *no callbacks*.

I also remember that while on that quest for parallelisation a number of options were available, but one stood out. It was quick, easy to introduce and well thought of: the excellent [`gevent`](http://www.gevent.org/) library. I arrived at it by reading this lovely hands-on tutorial: [*gevent for the Working Python Developer*](http://sdiehl.github.io/gevent-tutorial/), written by an awesome community of users, a great introduction not only to gevent but to concurrency in general, and you most definitely should check it out.

I like the tutorial so much that I decided it would be a good template to follow when introducing *asyncio*.

Quick disclaimer, this is not a gevent vs. asyncio article, Nathan Road wrote [a great piece](http://www.getoffmalawn.com/blog/playing-with-asyncio) on what’s similar and dissimilar between the two if you’re interested.

I know you’re excited but before we dive in I’d like to quickly go over some concepts that may not be familiar at first.

**Update June 2018:** In Python 3.7 asyncio has gotten a few upgrades in its API, particularly around managing of tasks and event loops. I’ve updated the examples to encourage adoption as I believe it’s cleaner and more concise. If you cannot update to 3.7 there are versions of the examples for 3.6 and below available in the [GitHub repository for this article](https://github.com/yeraydiazdiaz/asyncio-ftwpd).

**Update May 2018:** some readers reported that the code examples were no longer compatible with recent versions of `aiohttp`. I have now updated the examples to work with the most recent version at the time of this writing 3.2.1.

**Update Feb 2017:** following some feedback I’ve decided to use 3.5 async/await syntax, I’ve updated the examples accordingly. If you’re interested the original 3.4 syntax examples are available in the [Github repo for this tutorial](https://github.com/yeraydiazdiaz/asyncio-ftwpd).


## Threads, loops, coroutines and futures

Threads are a common tool and most developers have heard of and used before. However *asyncio* uses quite different constructs: **event loops**, **coroutines** and **futures**.

- An [event loop](https://docs.python.org/dev/library/asyncio-eventloop.html) essentially manages and distributes the execution of different tasks. It registers them and handles distributing the flow of control between them.
- [Coroutines](https://docs.python.org/3.5/library/asyncio-task.html#coroutines) are special functions that work similarly to Python generators, on `await` they release the flow of control back to the event loop. A coroutine needs to be scheduled to run on the event loop, once scheduled coroutines are wrapped in **Tasks** which is a type of **Future**.
- [Futures](https://docs.python.org/3.5/library/asyncio-task.html#future) are objects that represent the result of a task that may or may not have been executed. This result may be an exception.

Got it? Pretty simple, right? let’s dive right in!

## Synchronous & Asynchronous Execution

In [*Concurrency is not parallelism*](https://vimeo.com/49718712), it’s better Rob Pike makes a point that really made things click in my head. Breaking down tasks into concurrent subtasks only *allows* parallelism, it’s the scheduling of these subtasks that creates it.

*Asyncio* does exactly that, you can structure your code so subtasks are defined as coroutines and allows you to schedule them as you please, including simultaneously. Coroutines contain yield points where we define possible points where a context switch can happen if other tasks are pending, but will not if no other task is pending.

A context switch in *asyncio* represents the event loop yielding the flow of control from one coroutine to the next. Let’s have a look at a very basic example:

{% highlight python lineos %}
import asyncio


async def foo():
    print('Running in foo')
    await asyncio.sleep(0)
    print('Explicit context switch to foo again')


async def bar():
    print('Explicit context to bar')
    await asyncio.sleep(0)
    print('Implicit context switch back to bar')


async def main():
    tasks = [foo(), bar()]
    await asyncio.gather(*tasks)


asyncio.run(main())
{% endhighlight %}

```
$ python 1-sync-async-execution.py
Running in foo
Explicit context to bar
Explicit context switch to foo again
Implicit context switch back to bar
```

- First we declare a couple of simple coroutines that pretend to do non-blocking work using the `sleep` function in *asyncio*.
- Then we create an entry point coroutine from which we combine the previous coroutines using [`gather`](https://docs.python.org/3.7/library/asyncio-task.html#asyncio.gather) to wait for both of them to complete. There’s a bit more to `gather` than that but we’ll ignore it for now.
- And finally we schedule our entry point coroutine using [`asyncio.run`](https://docs.python.org/3.7/library/asyncio-task.html#asyncio.run), which will take care of creating an event loop and scheduling our entry point coroutine.

> Note that versions of Python prior to 3.7 coroutines had to be manually wrapped in Tasks to be scheduled using the current event loop’s [`create_task`](https://docs.python.org/3.7/library/asyncio-eventloop.html?highlight=create_task#asyncio.AbstractEventLoop.create_task) method. There was also a bit of boilerplate required to create an event loop and schedule our tasks. Please refer to the [GitHub repository](https://github.com/yeraydiazdiaz/asyncio-ftwpd) for code samples using these techniques.

By using `await` on another coroutine we declare that the coroutine may give the control back to the event loop, in this case `sleep`. The coroutine will yield and the event loop will switch contexts to the next task scheduled for execution: `bar`. Similarly the bar coroutine uses `await sleep` which allows the event loop to pass control back to foo at the point where it yielded before, just as normal Python generators.

Let’s now simulate two blocking tasks, `gr1` and `gr2`, say they’re two requests to external services. While those are executing a third task can be doing work asynchronously, like in the following example:

{% highlight python lineos %}
import time
import asyncio

start = time.time()


def tic():
    return 'at %1.1f seconds' % (time.time() - start)


async def gr1():
    # Busy waits for a second, but we don't want to stick around...
    print('gr1 started work: {}'.format(tic()))
    await asyncio.sleep(2)
    print('gr1 ended work: {}'.format(tic()))


async def gr2():
    # Busy waits for a second, but we don't want to stick around...
    print('gr2 started work: {}'.format(tic()))
    await asyncio.sleep(2)
    print('gr2 Ended work: {}'.format(tic()))


async def gr3():
    print("Let's do some stuff while the coroutines are blocked, {}".format(tic()))
    await asyncio.sleep(1)
    print("Done!")


async def main():
    tasks = [gr1(), gr2(), gr3()]
    await asyncio.gather(*tasks)


asyncio.run(main())
{% endhighlight %}

```
$ python 1b-cooperatively-scheduled.py
gr1 started work: at 0.0 seconds
gr2 started work: at 0.0 seconds
Let's do some stuff while the coroutines are blocked, at 0.0 seconds
Done!
gr1 ended work: at 2.0 seconds
gr2 Ended work: at 2.0 seconds
```

Notice how the event loop manages and schedules the execution allowing our single threaded code to operate concurrently. While the two blocking tasks are blocked a third one can take control of the flow.

## Order of execution

In the synchronous world we’re used to thinking linearly. If we were to have a series of tasks that take different amounts of time they will be executed in the order that they were called upon.

However, when using concurrency we need to be aware that the tasks finish in different order than they were scheduled.

{% highlight python lineos %}
import random
from time import sleep
import asyncio


def task(pid):
    """Synchronous non-deterministic task."""
    sleep(random.randint(0, 2) * 0.001)
    print('Task %s done' % pid)


async def task_coro(pid):
    """Coroutine non-deterministic task"""
    await asyncio.sleep(random.randint(0, 2) * 0.001)
    print('Task %s done' % pid)


def synchronous():
    for i in range(1, 10):
        task(i)


async def asynchronous():
    tasks = [task_coro(i) for i in range(1, 10)]
    await asyncio.gather(*tasks)


print('Synchronous:')
synchronous()

print('Asynchronous:')
asyncio.run(asynchronous())
{% endhighlight %}

```
$ python 1c-determinism-sync-async.py
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
Task 4 done
Task 9 done
Task 2 done
Task 3 done
Task 5 done
Task 6 done
Task 8 done
Task 1 done
Task 7 done
```

Your output will, of course, vary since each task will sleep for a random amount of time, but notice how the resulting order is completely different, even though we built the array of tasks in the same order using `range`.

It’s important to understand that *asyncio* does not magically make things non-blocking. At the time of writing *asyncio* stands alone in the standard library, the rest of modules provide only blocking functionality. You can use the [`concurrent.futures`](https://docs.python.org/3/library/concurrent.futures.html#module-concurrent.futures) module to wrap a blocking task in a thread or a process and return a `Future` *asyncio* can use. This same example using threads is available in the [Github repo](https://github.com/yeraydiazdiaz/asyncio-ftwpd).

This is probably the main drawback right now when using asyncio, however [there are plenty of libraries for different tasks and services](https://github.com/timofurrer/awesome-asyncio).

A very common blocking task is, of course, fetching data from an HTTP service. I’m using the excellent [`aiohttp`](https://aiohttp.readthedocs.org/) library for non-blocking HTTP requests retrieving data from Github’s public event API and simply take the `Date` response header.

**Please do not focus on the details of the aiohttp_get coroutines below**. They use [asynchronous context manager syntax](https://www.python.org/dev/peps/pep-0492/) which is outside the scope of this article but is [necessary boilerplate](http://aiohttp.readthedocs.io/en/stable/client_quickstart.html#make-a-request) to perform an asynchronous HTTP request using `aiohttp`. Just pretend is an external coroutine and focus on how it’s used below.

{% highlight python lineos %}
import time
import urllib.request
import asyncio
import aiohttp

URL = 'https://api.github.com/events'
MAX_CLIENTS = 3


def fetch_sync(pid):
    print('Fetch sync process {} started'.format(pid))
    start = time.time()
    response = urllib.request.urlopen(URL)
    datetime = response.getheader('Date')

    print('Process {}: {}, took: {:.2f} seconds'.format(
        pid, datetime, time.time() - start))

    return datetime


async def aiohttp_get(url):
    """Nothing to see here, carry on ..."""
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return response


async def fetch_async(pid):
    print('Fetch async process {} started'.format(pid))
    start = time.time()
    response = await aiohttp_get(URL)
    datetime = response.headers.get('Date')

    print('Process {}: {}, took: {:.2f} seconds'.format(
        pid, datetime, time.time() - start))

    response.close()
    return datetime


def synchronous():
    start = time.time()
    for i in range(1, MAX_CLIENTS + 1):
        fetch_sync(i)
    print("Process took: {:.2f} seconds".format(time.time() - start))


async def asynchronous():
    start = time.time()
    tasks = [asyncio.create_task(
        fetch_async(i)) for i in range(1, MAX_CLIENTS + 1)]
    await asyncio.wait(tasks)
    print("Process took: {:.2f} seconds".format(time.time() - start))


print('Synchronous:')
synchronous()

print('Asynchronous:')
asyncio.run(asynchronous())
{% endhighlight %}

```
$ python 1d-async-fetch-from-server.py
Synchronous:
Fetch sync process 1 started
Process 1: Fri, 29 Jun 2018 11:41:37 GMT, took: 0.76 seconds
Fetch sync process 2 started
Process 2: Fri, 29 Jun 2018 11:41:37 GMT, took: 0.67 seconds
Fetch sync process 3 started
Process 3: Fri, 29 Jun 2018 11:41:38 GMT, took: 0.68 seconds
Process took: 2.11 seconds
Asynchronous:
Fetch async process 1 started
Fetch async process 2 started
Fetch async process 3 started
Process 1: Fri, 29 Jun 2018 11:41:39 GMT, took: 0.70 seconds
Process 2: Fri, 29 Jun 2018 11:41:39 GMT, took: 0.71 seconds
Process 3: Fri, 29 Jun 2018 11:41:39 GMT, took: 0.84 seconds
Process took: 0.86 seconds
```

First off, note the difference in timing, by using asynchronous calls we’re making *at the same time* all the requests to the service. As discussed each request yields the control flow to the next and returns when it’s completed. The result is that requesting and retrieving the result of all requests takes only as long as the slowest request! See how the timing logs 0.84 seconds for the slowest request which is the about the total time elapsed by processing all the requests. Pretty cool, huh?

Secondly, look at how similar the code is to the synchronous version! It’s essentially the same! The main differences are due to library implementation for performing the GET request and creating the tasks and waiting for them to finishing.

## Creating concurrency

So far we’ve been using a single method of creating and retrieving results from coroutines, creating a set of tasks and waiting for all of them to finish.

But coroutines can be scheduled to run or retrieve their results in different ways. Imagine a scenario where we need to process the results of the HTTP GET requests as soon as they arrive, the process is actually quite similar than in our previous example:

{% highlight python lineos %}
import time
import random
import asyncio
import aiohttp

URL = 'https://api.github.com/events'
MAX_CLIENTS = 3


async def aiohttp_get(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return response


async def fetch_async(pid):
    start = time.time()
    sleepy_time = random.randint(2, 5)
    print('Fetch async process {} started, sleeping for {} seconds'.format(
        pid, sleepy_time))

    await asyncio.sleep(sleepy_time)

    response = await aiohttp_get(URL)
    datetime = response.headers.get('Date')

    response.close()
    return 'Process {}: {}, took: {:.2f} seconds'.format(
        pid, datetime, time.time() - start)


async def main():
    start = time.time()
    futures = [fetch_async(i) for i in range(1, MAX_CLIENTS + 1)]
    for i, future in enumerate(asyncio.as_completed(futures)):
        result = await future
        print('{} {}'.format(">>" * (i + 1), result))

    print("Process took: {:.2f} seconds".format(time.time() - start))


asyncio.run(main())
{% endhighlight %}

```
$ python 2a-async-fetch-from-server-as-completed.py
Fetch async process 2 started, sleeping for 5 seconds
Fetch async process 3 started, sleeping for 4 seconds
Fetch async process 1 started, sleeping for 3 seconds
>> Process 1: Fri, 29 Jun 2018 11:44:19 GMT, took: 3.70 seconds
>>>> Process 3: Fri, 29 Jun 2018 11:44:20 GMT, took: 4.68 seconds
>>>>>> Process 2: Fri, 29 Jun 2018 11:44:21 GMT, took: 5.68 seconds
Process took: 5.68 seconds
```

Note the padding and the timing of each result call, they are scheduled at the same time, the results arrive out of order and we process them as soon as they do.

The code in this case is only slightly different, we’re gathering the coroutines into a list, each of them ready to be scheduled and executed. The [`as_completed`](https://docs.python.org/dev/library/asyncio-task.html#asyncio.as_completed) function returns an iterator that will yield a completed future as they come in. Now don’t tell me that’s not cool. By the way, `as_completed` is originally from the [`concurrent.futures`](https://docs.python.org/dev/library/concurrent.futures.html#module-functions) module.

Let’s get to another example, imagine you’re trying to get your IP address. There are similar services you can use to retrieve it but you’re not sure if they will be accessible at runtime. You don’t want to check each one sequentially, ew. You would send concurrent requests to each service and pick the first one that responds, right? Right!

Well, there’s one more way of scheduling tasks in asyncio, [`wait`](https://docs.python.org/dev/library/asyncio-task.html#asyncio.wait), which happens to have a parameter to do just that: `return_when`. But now we want to retrieve the results from the coroutine, so we can use the two sets of futures, `done` and `pending`.

> In this next example we’re going to use the pre Python 3.7 way of starting things off in asyncio to illustrate a point, please bear with me:

{% highlight python lineos %}
from collections import namedtuple
import time
import asyncio
from concurrent.futures import FIRST_COMPLETED
import aiohttp

Service = namedtuple('Service', ('name', 'url', 'ip_attr'))

SERVICES = (
    Service('ipify', 'https://api.ipify.org?format=json', 'ip'),
    Service('ip-api', 'http://ip-api.com/json', 'query')
)


async def aiohttp_get_json(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()


async def fetch_ip(service):
    start = time.time()
    print('Fetching IP from {}'.format(service.name))

    json_response = await aiohttp_get_json(service.url)
    ip = json_response[service.ip_attr]

    return '{} finished with result: {}, took: {:.2f} seconds'.format(
        service.name, ip, time.time() - start)


async def main():
    futures = [fetch_ip(service) for service in SERVICES]
    done, pending = await asyncio.wait(
        futures, return_when=FIRST_COMPLETED)

    print(done.pop().result())


ioloop = asyncio.get_event_loop()
ioloop.run_until_complete(main())
ioloop.close()
{% endhighlight%}

```
$ python 2b-fetch-first-ip-address-response-await.py
Fetching IP from ip-api
Fetching IP from ipify
ip-api finished with result: 81.106.46.223, took: 0.10 seconds
Task was destroyed but it is pending!
task: <Task pending coro=<fetch_ip() done, defined at 2b-fetch-first-ip-address-response-await.py:21> wait_for=<Future pending cb=[BaseSelectorEventLoop._sock_connect_done(10)(), <TaskWakeupMethWrapper object at 0x10c11cd38>()]>>
```

Wait, what happened there? The first service responded just fine but what’s with all those warnings?

Well, we scheduled two tasks but once the first one completed the closed the loop leaving the second one pending. *Asyncio* assumes that’s a bug and prints out a warning. We really should clean up after ourselves and let the event loop know not to bother with the pending futures. How? Glad you asked.

## Future states

(As in states that a Future can be in, not states that are in the future… you know what I mean)

These are:

- Pending
- Running
- Done
- Cancelled

As simple as that. When a future is done its result method will return the result of the future, if it’s pending or running it raises `InvalidStateError`, if it’s cancelled it will raise `CancelledError`, and finally if the coroutine raised an exception it will be raised again, which is the same behaviour as calling `exception`. [But don’t take my word for it](https://docs.python.org/dev/library/asyncio-task.html#future).

You can also call `done`, `cancelled` or `running` on a Future to get a boolean if the Future is in that state, note that `done` simply means `result` will return or raise an exception. You can specifically cancel a Future by calling the `cancel` method (oddly enough), which is exactly what `asyncio.run` does under the hood in Python 3.7 so you don’t have to worry about it.

{% highlight python lineos %}
from collections import namedtuple
import time
import asyncio
from concurrent.futures import FIRST_COMPLETED
import aiohttp

Service = namedtuple('Service', ('name', 'url', 'ip_attr'))

SERVICES = (
    Service('ipify', 'https://api.ipify.org?format=json', 'ip'),
    Service('ip-api', 'http://ip-api.com/json', 'query')
)


async def aiohttp_get_json(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()


async def fetch_ip(service):
    start = time.time()
    print('Fetching IP from {}'.format(service.name))

    json_response = await aiohttp_get_json(service.url)
    ip = json_response[service.ip_attr]

    return '{} finished with result: {}, took: {:.2f} seconds'.format(
        service.name, ip, time.time() - start)


async def main():
    futures = [fetch_ip(service) for service in SERVICES]
    done, pending = await asyncio.wait(
        futures, return_when=FIRST_COMPLETED)

    print(done.pop().result())


asyncio.run(main())
{% endhighlight%}

```
$ python 2c-fetch-first-ip-address-response-no-warning.py
Fetching IP from ip-api
Fetching IP from ipify
ip-api finished with result: 81.106.46.223, took: 0.12 seconds
```

Nice and tidy output, gotta love it.

This type of “Task is destroyed but is was pending” error is quite common when working with *asyncio* and now you know the reason behind it and how to avoid it, I hope you can forgive my little detour to pre-3.7 land.

Futures also allow attaching callbacks when they get to the done state in case you want to add additional logic. You can even manually set the result or the exception of a Future, typically for unit testing purposes.

## Exception handling

*Asyncio* is all about making concurrent code manageable and readable, and that becomes really obvious in the handling of exceptions. Let’s go back to an example to illustrate this.

Imagine we want to ensure all our IP services return the same result, but one of our services is offline and not resolving. We can simply use `try...except`, as usual:

{% highlight python lineos %}
from collections import namedtuple
import time
import asyncio
import aiohttp

Service = namedtuple('Service', ('name', 'url', 'ip_attr'))

SERVICES = (
    Service('ipify', 'https://api.ipify.org?format=json', 'ip'),
    Service('ip-api', 'http://ip-api.com/json', 'query'),
    Service('borken', 'http://no-way-this-is-going-to-work.com/json', 'ip')
)

async def aiohttp_get_json(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()


async def fetch_ip(service):
    start = time.time()
    print('Fetching IP from {}'.format(service.name))

    try:
        json_response = await aiohttp_get_json(service.url)
    except:
        return '{} is unresponsive'.format(service.name)

    ip = json_response[service.ip_attr]

    return '{} finished with result: {}, took: {:.2f} seconds'.format(
        service.name, ip, time.time() - start)


async def main():
    futures = [fetch_ip(service) for service in SERVICES]
    done, _ = await asyncio.wait(futures)

    for future in done:
        print(future.result())


asyncio.run(main())
{% endhighlight%}

```
$ python 3a-fetch-ip-addresses-fail.py
Fetching IP from ip-api
Fetching IP from ipify
Fetching IP from borken
ipify finished with result: 81.106.46.223, took: 5.35 seconds
borken is unresponsive
ip-api finished with result: 81.106.46.223, took: 4.91 seconds
```

We can also handle the exceptions as we process the results of the futures, in case an unexpected exception occurred:

{% highlight python lineos %}
from collections import namedtuple
import time
import asyncio
import aiohttp
import traceback

Service = namedtuple('Service', ('name', 'url', 'ip_attr'))

SERVICES = (
    Service('ipify', 'https://api.ipify.org?format=json', 'ip'),
    Service('ip-api', 'http://ip-api.com/json', 'this-is-not-an-attr'),
    Service('borken', 'http://no-way-this-is-going-to-work.com/json', 'ip')
)

async def aiohttp_get_json(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()


async def fetch_ip(service):
    start = time.time()
    print('Fetching IP from {}'.format(service.name))

    try:
        json_response = await aiohttp_get_json(service.url)
    except:
        return '{} is unresponsive'.format(service.name)

    ip = json_response[service.ip_attr]

    return '{} finished with result: {}, took: {:.2f} seconds'.format(
        service.name, ip, time.time() - start)


async def main():
    futures = [fetch_ip(service) for service in SERVICES]
    done, _ = await asyncio.wait(futures)

    for future in done:
        try:
            print(future.result())
        except:
            print("Unexpected error: {}".format(traceback.format_exc()))


asyncio.run(main())
{% endhighlight%}

```
$ python 3b-fetch-ip-addresses-future-exceptions.py
Fetching IP from ip-api
Fetching IP from ipify
Fetching IP from borken
borken is unresponsive
Unexpected error: Traceback (most recent call last):
  File "3b-fetch-ip-addresses-future-exceptions-await.py", line 42, in main
    print(future.result())
  File "3b-fetch-ip-addresses-future-exceptions-await.py", line 30, in fetch_ip
    ip = json_response[service.ip_attr]
KeyError: 'this-is-not-an-attr'

ipify finished with result: 81.106.46.223, took: 0.52 seconds
```

Didn’t see that one coming…

In the same way that scheduling a task and not waiting for it to finish is considered a bug, scheduling a task and not retrieving the possible exceptions raised will also throw a warning:

{% highlight python lineos %}
from collections import namedtuple
import time
import asyncio
import aiohttp

Service = namedtuple('Service', ('name', 'url', 'ip_attr'))

SERVICES = (
    Service('ipify', 'https://api.ipify.org?format=json', 'ip'),
    Service('ip-api', 'http://ip-api.com/json', 'this-is-not-an-attr'),
    Service('borken', 'http://no-way-this-is-going-to-work.com/json', 'ip')
)

async def aiohttp_get_json(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()


async def fetch_ip(service):
    start = time.time()
    print('Fetching IP from {}'.format(service.name))

    try:
        json_response = await aiohttp_get_json(service.url)
    except:
        print('{} is unresponsive'.format(service.name))
    else:
        ip = json_response[service.ip_attr]

        print('{} finished with result: {}, took: {:.2f} seconds'.format(
            service.name, ip, time.time() - start))


async def main():
    futures = [fetch_ip(service) for service in SERVICES]
    await asyncio.wait(futures)  # intentionally ignore results


asyncio.run(main())
{% endhighlight%}

```
$ python 3c-fetch-ip-addresses-ignore-exceptions-await.py
Fetching IP from borken
Fetching IP from ip-api
Fetching IP from ipify
borken is unresponsive
ipify finished with result: 81.106.46.223, took: 1.41 seconds
Task exception was never retrieved
future: <Task finished coro=<fetch_ip() done, defined at 3c-fetch-ip-addresses-ignore-exceptions-await.py:20> exception=KeyError('this-is-not-an-attr')>
Traceback (most recent call last):
  File "3c-fetch-ip-addresses-ignore-exceptions-await.py", line 29, in fetch_ip
    ip = json_response[service.ip_attr]
KeyError: 'this-is-not-an-attr'
```

That looks remarkably like the output from our previous example, minus the tut-tut message from *asyncio*.

## Timeouts

What if we don’t really care that much about our IP? Imagine it being a nice addition to a more complex response but we certainly don’t want to keep the user waiting for it. Ideally we’d give our non-blocking calls a timeout, after which we just send our complex response without the IP attribute.

Again `wait` has just the attribute we need:

{% highlight python lineos %}
import time
import random
import asyncio
import aiohttp
import argparse
from collections import namedtuple
from concurrent.futures import FIRST_COMPLETED

Service = namedtuple('Service', ('name', 'url', 'ip_attr'))

SERVICES = (
    Service('ipify', 'https://api.ipify.org?format=json', 'ip'),
    Service('ip-api', 'http://ip-api.com/json', 'query'),
)

DEFAULT_TIMEOUT = 0.01


async def aiohttp_get_json(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()


async def fetch_ip(service):
    start = time.time()
    print('Fetching IP from {}'.format(service.name))

    await asyncio.sleep(random.randint(1, 3) * 0.1)
    try:
        json_response = await aiohttp_get_json(service.url)
    except:
        return '{} is unresponsive'.format(service.name)

    ip = json_response[service.ip_attr]

    print('{} finished with result: {}, took: {:.2f} seconds'.format(
        service.name, ip, time.time() - start))
    return ip


async def main(timeout):
    response = {
        "message": "Result from asynchronous.",
        "ip": "not available"
    }

    futures = [fetch_ip(service) for service in SERVICES]
    done, pending = await asyncio.wait(
        futures, timeout=timeout, return_when=FIRST_COMPLETED)

    for future in pending:
        future.cancel()

    for future in done:
        response["ip"] = future.result()

    print(response)


parser = argparse.ArgumentParser()
parser.add_argument(
    '-t', '--timeout',
    help='Timeout to use, defaults to {}'.format(DEFAULT_TIMEOUT),
    default=DEFAULT_TIMEOUT, type=float)
args = parser.parse_args()

print("Using a {} timeout".format(args.timeout))
asyncio.run(main(args.timeout))
{% endhighlight%}

Notice the `timeout` argument on `wait`, we’re also adding a command line argument to test what happens if we do allow the requests some time. I also added a some random sleeping time to ensure things didn’t move too fast.

```
$ python 4a-timeout-with-wait-kwarg-await.py
Using a 0.01 timeout
Fetching IP from ipify
Fetching IP from ip-api
{'message': 'Result from asynchronous.', 'ip': 'not available'}

$ python 4a-timeout-with-wait-kwarg-await.py -t 5
Using a 5.0 timeout
Fetching IP from ipify
Fetching IP from ip-api
ip-api finished with result: 81.106.46.223, took: 0.22 seconds
{'message': 'Result from asynchronous.', 'ip': '81.106.46.223'}
```

## Conclusion

*Asyncio* has extended my already ample love for Python. To be absolutely honest I fell in love with marriage of coroutines and Python when I first discovered [*Tornado*](http://tornadoweb.org/) but *asyncio* has managed to unify the best of this and the rest of excellent concurrency libraries into a rock solid piece. So much so that a special effort was made to ensure these and other libraries can use the main IO loop, so if you’re using *Tornado* or [*Twisted*](https://www.twistedmatrix.com/) you can make use of libraries intended for *asyncio*!

As I said before its main problem is the lack of standard library modules that implement non-blocking behaviour. You may find that a particular technology that has plenty of well established Python libraries to interact with will not have a non-blocking version, or the existing ones are young lived or experimental. However, [the number asyncio compatible libraries always increasing](https://github.com/timofurrer/awesome-asyncio).

Hopefully in this tutorial I communicated what a joy is to work with asyncio. I honestly think it’s the piece that will finally make adaptation to Python 3 a reality, it really feels you’re missing out if you’re stuck with Python 2.7. One thing’s for sure, Python’s future has completely changed, pun intended.

P.S. If you want more asyncio goodness I’ve written a two-part follow up article to this one: [*Asyncio Coroutine Patterns: Beyond await*]({% post_url 2017-06-03-asyncio-coroutine-patterns %}) and [*Asyncio Coroutine Patterns: Errors and Cancellation*]({% post_url 2017-08-29-asyncio-coroutine-patterns-errors-and-cancellation %}), happy awaiting!
</div>