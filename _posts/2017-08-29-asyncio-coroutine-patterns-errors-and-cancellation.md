---
layout: post
title:  "Asyncio Coroutine Patterns: Errors and Cancellation"
date:   2017-08-29 14:08:51 +0100
categories: python asyncio
tags: python asyncio python3
image: asyncio-coroutine-patterns-errors-and-cancellation.jpg
short_description: Dig deeper into Python's concurrency library, asyncio. This tutorial covers errors and cancellation of tasks and futures with a hands on example performing concurrent HTTP requests.
keywords: "python, tutorial, asyncio, concurrency, errors, futures, cancellation, task, coroutine"
canonical_url: https://medium.com/@yeraydiazdiaz/asyncio-coroutine-patterns-errors-and-cancellation-3bb422e961ff
---

<div markdown="1" class="sticky">
* TOC
{:toc}
</div>

<div markdown="1" id="text">
This is the second part of a two part series on coroutine patterns in *asyncio*, to fully benefit from this article please read the first installment: [*Asyncio Coroutine Patterns: Beyond await*]({% post_url 2017-06-03-asyncio-coroutine-patterns %})

In the first part of this series we concluded that *asyncio* is awesome, coroutines are awesome and our code is awesome. But sometimes the outside world is not as awesome and we have to deal with it.

Now, for this second part of the series, I'll run over the options *asyncio* gives us to handle errors when using these patterns and cancelling tasks so as to make our asynchronous systems robust and performant.

If you are really new to *asyncio* I recommend having a read through my very first article on the subject, [*Asyncio for the Working Python Developer*]({% post_url 2016-02-20-asyncio-for-the-working-python-developer %}), before diving into this series.

<!--more-->

<div class="note">
<a href="{{ page.canonical_url }}" target="_blank"><em>Asyncio Coroutine Patterns: Errors and Cancellation</em> was first published on Medium</a>, if you'd like to comment or give feedback please do so there.
</div>

I will be continuing with the examples using the [Hacker News API](https://github.com/HackerNews/API) and also be using the async/await syntax introduced Python 3.5+ and all example code is available in the [Github repo for this series](https://github.com/yeraydiazdiaz/asyncio-coroutine-patterns).

Let’s get to it!

## Error handling

If you recall at the end of the previous article we had a lovely system using periodic task that fetched the top stories in HN and recursively calculates the number of comments for each of them. Here’s the full listing:

{% highlight python lineos %}
"""
An example of periodically scheduling coroutines using an infinite loop of
scheduling a task using ensure_future and sleeping.
Artificially produce an error on URLFetcher.
"""

import asyncio
import argparse
import logging
from datetime import datetime

import aiohttp
import async_timeout


LOGGER_FORMAT = '%(asctime)s %(message)s'
URL_TEMPLATE = "https://hacker-news.firebaseio.com/v0/item/{}.json"
TOP_STORIES_URL = "https://hacker-news.firebaseio.com/v0/topstories.json"
FETCH_TIMEOUT = 10
MAXIMUM_FETCHES = 5

parser = argparse.ArgumentParser(
    description='Calculate the number of comments of the top stories in HN.')
parser.add_argument(
    '--period', type=int, default=5, help='Number of seconds between poll')
parser.add_argument(
    '--limit', type=int, default=5,
    help='Number of new stories to calculate comments for')
parser.add_argument('--verbose', action='store_true', help='Detailed output')


logging.basicConfig(format=LOGGER_FORMAT, datefmt='[%H:%M:%S]')
log = logging.getLogger()
log.setLevel(logging.INFO)


class BoomException(Exception):
    pass


class URLFetcher():
    """Provides counting of URL fetches for a particular task.
    """

    def __init__(self):
        self.fetch_counter = 0

    async def fetch(self, session, url):
        """Fetch a URL using aiohttp returning parsed JSON response.
        As suggested by the aiohttp docs we reuse the session.
        """
        with async_timeout.timeout(FETCH_TIMEOUT):
            self.fetch_counter += 1
            if self.fetch_counter > MAXIMUM_FETCHES:
                raise BoomException('BOOM!')

            async with session.get(url) as response:
                return await response.json()


async def post_number_of_comments(loop, session, fetcher, post_id):
    """Retrieve data for current post and recursively for all comments.
    """
    url = URL_TEMPLATE.format(post_id)
    response = await fetcher.fetch(session, url)

    # base case, there are no comments
    if response is None or 'kids' not in response:
        return 0

    # calculate this post's comments as number of comments
    number_of_comments = len(response['kids'])

    # create recursive tasks for all comments
    tasks = [post_number_of_comments(
        loop, session, fetcher, kid_id) for kid_id in response['kids']]

    # schedule the tasks and retrieve results
    results = await asyncio.gather(*tasks)

    # reduce the descendents comments and add it to this post's
    number_of_comments += sum(results)
    log.debug('{:^6} > {} comments'.format(post_id, number_of_comments))

    return number_of_comments


async def get_comments_of_top_stories(loop, session, limit, iteration):
    """Retrieve top stories in HN.
    """
    fetcher = URLFetcher()  # create a new fetcher for this task
    response = await fetcher.fetch(session, TOP_STORIES_URL)

    tasks = [post_number_of_comments(
        loop, session, fetcher, post_id) for post_id in response[:limit]]

    results = await asyncio.gather(*tasks)

    for post_id, num_comments in zip(response[:limit], results):
        log.info("Post {} has {} comments ({})".format(
            post_id, num_comments, iteration))
    return fetcher.fetch_counter  # return the fetch count


async def poll_top_stories_for_comments(loop, session, period, limit):
    """Periodically poll for new stories and retrieve number of comments.
    """
    iteration = 1
    while True:
        log.info("Calculating comments for top {} stories. ({})".format(
            limit, iteration))

        future = asyncio.ensure_future(
            get_comments_of_top_stories(loop, session, limit, iteration))

        now = datetime.now()

        def callback(fut):
            fetch_count = fut.result()
            log.info(
                '> Calculating comments took {:.2f} seconds and {} fetches'.format(
                    (datetime.now() - now).total_seconds(), fetch_count))

        future.add_done_callback(callback)

        log.info("Waiting for {} seconds...".format(period))
        iteration += 1
        await asyncio.sleep(period)


async def main(loop, period, limit):
    """Async entry point coroutine.
    """
    async with aiohttp.ClientSession(loop=loop) as session:
        comments = await poll_top_stories_for_comments(loop, session, period, limit)

    return comments


if __name__ == '__main__':
    args = parser.parse_args()
    if args.verbose:
        log.setLevel(logging.DEBUG)

    loop = asyncio.get_event_loop()
    loop.run_until_complete(main(loop, args.period, args.limit))

    loop.close()
{% endhighlight %}

Notice how `URLFetcher` raises a `BoomException` after a set number of fetches and there’s no exception handling anywhere in the code.

Here’s our periodic coroutine one more time:

{% highlight python lineos %}
async def poll_top_stories_for_comments(loop, session, period, limit):
    """Periodically poll for new stories and retrieve number of comments.
    """
    iteration = 1
    while True:
        log.info("Calculating comments for top {} stories. ({})".format(
            limit, iteration))

        future = asyncio.ensure_future(
            get_comments_of_top_stories(loop, session, limit, iteration))

        now = datetime.now()

        def callback(fut):
            fetch_count = fut.result()
            log.info(
                '> Calculating comments took {:.2f} seconds and {} fetches'.format(
                    (datetime.now() - now).total_seconds(), fetch_count))

        future.add_done_callback(callback)

        log.info("Waiting for {} seconds...".format(period))
        iteration += 1
        await asyncio.sleep(period)
{% endhighlight %}

When we run the script we get something like:

```
[16:13:16] Calculating comments for top 5 stories. (1)
[16:13:16] Waiting for 5 seconds...
[16:13:17] Exception in callback poll_top_stories_for_comments.<locals>.callback(<Task finishe...ion('BOOM!',)>) at 01_error_handling.py:126
handle: <Handle poll_top_stories_for_comments.<locals>.callback(<Task finishe...ion('BOOM!',)>) at 01_error_handling.py:126>
Traceback (most recent call last):
  File "/Users/yeray/.pyenv/versions/3.6.0/lib/python3.6/asyncio/events.py", line 126, in _run
    self._callback(*self._args)
  File "01_error_handling.py", line 127, in callback
    fetch_count = fut.result()
  File "01_error_handling.py", line 104, in get_comments_of_top_stories
    results = await asyncio.gather(*tasks)
  File "01_error_handling.py", line 71, in post_number_of_comments
    response = await fetcher.fetch(session, url)
  File "01_error_handling.py", line 60, in fetch
    raise BoomException('BOOM!')
BoomException: BOOM!
[16:13:21] Calculating comments for top 5 stories. (2)
[16:13:21] Waiting for 5 seconds...
...
```

See how the system **did not crash**? Our Tasks completed and when we retrieved their *result* in the callback an unhandled exception was raised. Usually this would have caused the Python interpreter to stop but it simply continued on attempting to fetch posts and comments a second time.

This is an important point, when using `ensure_future`, **exceptions will not crash the system and might go unnoticed**. So you need to ensure you’re handling exceptions and logging them appropriately.

Let’s change our periodic coroutine to handle this exception, remember since we’re using the `Future.result()` method to retrieve the result of the Task it will raise the exception if that was its result:

{% highlight python lineos %}
async def poll_top_stories_for_comments(loop, session, period, limit):
    """Periodically poll for new stories and retrieve number of comments.
    """
    iteration = 1
    while True:
        log.info("Calculating comments for top {} stories. ({})".format(
            limit, iteration))

        future = asyncio.ensure_future(
            get_comments_of_top_stories(loop, session, limit, iteration))

        now = datetime.now()

        def callback(fut):
            try:
                fetch_count = fut.result()
            except BoomException:
                log.exception("Something went BOOM")
                return

            log.info(
                '> Calculating comments took {:.2f} seconds and {} fetches'.format(
                    (datetime.now() - now).total_seconds(), fetch_count))

        future.add_done_callback(callback)

        log.info("Waiting for {} seconds...".format(period))
        iteration += 1
        await asyncio.sleep(period)
{% endhighlight %}

```
[16:21:03] Calculating comments for top 5 stories. (1)
[16:21:03] Waiting for 5 seconds…
[16:21:04] Something went BOOM
Traceback (most recent call last):
 File “01b_error_handling.py”, line 128, in callback
 fetch_count = fut.result()
 File “01b_error_handling.py”, line 104, in get_comments_of_top_stories
 results = await asyncio.gather(*tasks)
 File “01b_error_handling.py”, line 71, in post_number_of_comments
 response = await fetcher.fetch(session, url)
 File “01b_error_handling.py”, line 60, in fetch
 raise BoomException(‘BOOM!’)
BoomException: BOOM!
[16:21:08] Calculating comments for top 5 stories. (2)
[16:21:08] Waiting for 5 seconds…
[16:21:08] Something went BOOM
Traceback (most recent call last):
 File “01b_error_handling.py”, line 128, in callback
 fetch_count = fut.result()
 File “01b_error_handling.py”, line 104, in get_comments_of_top_stories
 results = await asyncio.gather(*tasks)
 File “01b_error_handling.py”, line 71, in post_number_of_comments
 response = await fetcher.fetch(session, url)
 File “01b_error_handling.py”, line 60, in fetch
 raise BoomException(‘BOOM!’)
BoomException: BOOM!
```

This has not completely solved the issue as the system will still not crash, but at least the exception is being logged appropriately.

However, as good practice dictates, we’re catching a specific type of exceptions. What if some other type of exception occurs? Let’s simulate this with as small change in `URLFetcher`:

{% highlight python lineos %}
class URLFetcher():
    """Provides counting of URL fetches for a particular task.
    """

    def __init__(self):
        self.fetch_counter = 0

    async def fetch(self, session, url):
        """Fetch a URL using aiohttp returning parsed JSON response.
        As suggested by the aiohttp docs we reuse the session.
        """
        with async_timeout.timeout(FETCH_TIMEOUT):
            self.fetch_counter += 1
            if self.fetch_counter > MAXIMUM_FETCHES:
                raise BoomException('BOOM!')
            elif randint(0, 3) == 0:
                raise Exception('Random generic exception')

            async with session.get(url) as response:
                return await response.json()
{% endhighlight %}

```
[16:23:22] Calculating comments for top 5 stories. (1)
[16:23:22] Waiting for 5 seconds…
[16:23:22] Exception in callback poll_top_stories_for_comments.<locals>.callback(<Task finishe… exception’,)>) at 01c_error_handling.py:129
handle: <Handle poll_top_stories_for_comments.<locals>.callback(<Task finishe… exception’,)>) at 01c_error_handling.py:129>
Traceback (most recent call last):
 File “/Users/yeray/.pyenv/versions/3.6.0/lib/python3.6/asyncio/events.py”, line 126, in _run
 self._callback(*self._args)
 File “01c_error_handling.py”, line 131, in callback
 fetch_count = fut.result()
 File “01c_error_handling.py”, line 107, in get_comments_of_top_stories
 results = await asyncio.gather(*tasks)
 File “01c_error_handling.py”, line 74, in post_number_of_comments
 response = await fetcher.fetch(session, url)
 File “01c_error_handling.py”, line 63, in fetch
 raise Exception(‘Random generic exception’)
Exception: Random generic exception
```

Again producing a silent unlogged error in our system.

The key points here are:

1. when using `await` handle exceptions with `try..except` as usual.
2. when using `ensure_future` remember to **catch generic exceptions** and act accordingly.

Here’s a complete listing of the system handling both cases:

{% highlight python lineos %}
"""
An example of periodically scheduling coroutines using an infinite loop of
scheduling a task using ensure_future and sleeping.
Artificially produce an error and use try..except clauses to catch them.
"""

import asyncio
import argparse
import logging
from datetime import datetime
from random import randint

import aiohttp
import async_timeout


LOGGER_FORMAT = '%(asctime)s %(message)s'
URL_TEMPLATE = "https://hacker-news.firebaseio.com/v0/item/{}.json"
TOP_STORIES_URL = "https://hacker-news.firebaseio.com/v0/topstories.json"
FETCH_TIMEOUT = 10
MAXIMUM_FETCHES = 5

parser = argparse.ArgumentParser(
    description='Calculate the number of comments of the top stories in HN.')
parser.add_argument(
    '--period', type=int, default=5, help='Number of seconds between poll')
parser.add_argument(
    '--limit', type=int, default=5,
    help='Number of new stories to calculate comments for')
parser.add_argument('--verbose', action='store_true', help='Detailed output')


logging.basicConfig(format=LOGGER_FORMAT, datefmt='[%H:%M:%S]')
log = logging.getLogger()
log.setLevel(logging.INFO)


class BoomException(Exception):
    pass


class URLFetcher():
    """Provides counting of URL fetches for a particular task.
    """

    def __init__(self):
        self.fetch_counter = 0

    async def fetch(self, session, url):
        """Fetch a URL using aiohttp returning parsed JSON response.
        As suggested by the aiohttp docs we reuse the session.
        """
        with async_timeout.timeout(FETCH_TIMEOUT):
            self.fetch_counter += 1
            if self.fetch_counter > MAXIMUM_FETCHES:
                raise BoomException('BOOM!')
            elif randint(0, 3) == 0:
                raise Exception('Random generic exception')

            async with session.get(url) as response:
                return await response.json()


async def post_number_of_comments(loop, session, fetcher, post_id):
    """Retrieve data for current post and recursively for all comments.
    """
    url = URL_TEMPLATE.format(post_id)
    try:
        response = await fetcher.fetch(session, url)
    except BoomException as e:
        log.error("Error retrieving post {}: {}".format(post_id, e))
        raise e

    # base case, there are no comments
    if response is None or 'kids' not in response:
        return 0

    # calculate this post's comments as number of comments
    number_of_comments = len(response['kids'])

    # create recursive tasks for all comments
    tasks = [post_number_of_comments(
        loop, session, fetcher, kid_id) for kid_id in response['kids']]

    # schedule the tasks and retrieve results
    try:
        results = await asyncio.gather(*tasks)
    except BoomException as e:
        log.error("Error retrieving comments for top stories: {}".format(e))
        raise

    # reduce the descendents comments and add it to this post's
    number_of_comments += sum(results)
    log.debug('{:^6} > {} comments'.format(post_id, number_of_comments))

    return number_of_comments


async def get_comments_of_top_stories(loop, session, limit, iteration):
    """Retrieve top stories in HN.
    """
    fetcher = URLFetcher()  # create a new fetcher for this task
    try:
        response = await fetcher.fetch(session, TOP_STORIES_URL)
    except Exception as e:
        log.error("Error retrieving top stories: {}".format(e))
        raise

    tasks = [post_number_of_comments(
        loop, session, fetcher, post_id) for post_id in response[:limit]]

    try:
        results = await asyncio.gather(*tasks)
    except BoomException as e:
        log.error("Error retrieving comments for top stories: {}".format(e))
        raise

    for post_id, num_comments in zip(response[:limit], results):
        log.info("Post {} has {} comments ({})".format(
            post_id, num_comments, iteration))
    return fetcher.fetch_counter  # return the fetch count


async def poll_top_stories_for_comments(loop, session, period, limit):
    """Periodically poll for new stories and retrieve number of comments.
    """
    iteration = 1
    while True:
        log.info("Calculating comments for top {} stories. ({})".format(
            limit, iteration))

        future = asyncio.ensure_future(
            get_comments_of_top_stories(loop, session, limit, iteration))

        now = datetime.now()

        def callback(fut):
            try:
                fetch_count = fut.result()
            except BoomException:
                pass  # a handled exception
            except Exception as e:
                log.exception('Unexpected error')
            else:
                log.info(
                    '> Calculating comments took {:.2f} seconds and {} fetches'.format(
                        (datetime.now() - now).total_seconds(), fetch_count))

        future.add_done_callback(callback)

        log.info("Waiting for {} seconds...".format(period))
        iteration += 1
        await asyncio.sleep(period)


async def main(loop, period, limit):
    """Async entry point coroutine.
    """
    async with aiohttp.ClientSession(loop=loop) as session:
        comments = await poll_top_stories_for_comments(loop, session, period, limit)

    return comments


if __name__ == '__main__':
    args = parser.parse_args()
    if args.verbose:
        log.setLevel(logging.DEBUG)

    loop = asyncio.get_event_loop()
    loop.run_until_complete(main(loop, args.period, args.limit))

    loop.close()
{% endhighlight %}

Ok, so given that we’ve been hitting errors from the HN API we should probably stop the system, otherwise it’s just going to keep raising error after error.

We can do it the hard way and add `loop.stop()` to our handling of errors in the callback code:

{% highlight python lineos %}
async def poll_top_stories_for_comments(loop, session, period, limit):
    """Periodically poll for new stories and retrieve number of comments.
    """
    iteration = 1
    while True:
        log.info("Calculating comments for top {} stories. ({})".format(
            limit, iteration))

        future = asyncio.ensure_future(
            get_comments_of_top_stories(loop, session, limit, iteration))

        now = datetime.now()

        def callback(fut):
            try:
                fetch_count = fut.result()
            except BoomException:
                loop.stop()
            except Exception as e:
                log.exception('Unexpected error')
                loop.stop()
            else:
                log.info(
                    '> Calculating comments took {:.2f} seconds and {} fetches'.format(
                        (datetime.now() - now).total_seconds(), fetch_count))

        future.add_done_callback(callback)

        log.info("Waiting for {} seconds...".format(period))
        iteration += 1
        await asyncio.sleep(period)
{% endhighlight %}

```
[16:34:00] Calculating comments for top 5 stories. (1)
[16:34:00] Waiting for 5 seconds...
[16:34:01] Error retrieving post 15052691: BOOM!
[16:34:01] Error retrieving comments for top stories: BOOM!
Traceback (most recent call last):
  File "02b_error_handling.py", line 175, in <module>
    loop, session, args.period, args.limit))
  File "/Users/yeray/.pyenv/versions/3.6.0/lib/python3.6/asyncio/base_events.py", line 464, in run_until_complete
    raise RuntimeError('Event loop stopped before Future completed.')
RuntimeError: Event loop stopped before Future completed.
```

Well, that stopped it all right, but *asyncio* did not like it, we’re forcibly closing the loop while other Futures are still pending, not cool.

In order to exit cleanly we need to **return from the periodic coroutine** which the loop will detect as complete and close itself.

However the exception is handled inside a callback, but since the callback is triggered independently we need some way of reading the error from the main loop and act accordingly.

We can do that by defining a list of errors in the enclosing scope and mutating it in the callback so we can exit in the next iteration if there are any:

{% highlight python lineos %}
async def poll_top_stories_for_comments(loop, session, period, limit):
    """Periodically poll for new stories and retrieve number of comments.
    """
    iteration = 1
    errors = []
    while True:
        if errors:
            print('Error detected, quitting')
            return

        log.info("Calculating comments for top {} stories. ({})".format(
            limit, iteration))

        future = asyncio.ensure_future(
            get_comments_of_top_stories(loop, session, limit, iteration))

        now = datetime.now()

        def callback(fut, errors):
            try:
                fetch_count = fut.result()
            except BoomException as e:
                log.debug('Adding {} to errors'.format(e))
                errors.append(e)
            except Exception as e:
                log.exception('Unexpected error')
                errors.append(e)
            else:
                log.info(
                    '> Calculating comments took {:.2f} seconds and {} fetches'.format(
                        (datetime.now() - now).total_seconds(), fetch_count))

        future.add_done_callback(partial(callback, errors=errors))

        log.info("Waiting for {} seconds...".format(period))
        iteration += 1
        await asyncio.sleep(period)
{% endhighlight %}

```
[16:48:59] Calculating comments for top 5 stories. (1)
[16:48:59] Waiting for 5 seconds...
[16:48:59] Error retrieving top stories: Random generic exception
[16:48:59] Unexpected error
Traceback (most recent call last):
  File "02c_error_handling.py", line 153, in callback
    fetch_count = fut.result()
  File "02c_error_handling.py", line 112, in get_comments_of_top_stories
    response = await fetcher.fetch(session, TOP_STORIES_URL)
  File "02c_error_handling.py", line 64, in fetch
    raise Exception('Random generic exception')
Exception: Random generic exception
[16:49:04] Error detected, quitting
```

I know, not pretty, and notice the timestamp as well, the return happened after the sleep which is not ideal but at least we get a clean exit from the loop.

Remember this happens only when using `ensure_future`. You can always go back to using `await` as described my first article if this is unacceptable.

## To raise or not to raise

Let’s step back for a moment though. In our example we’re being quite extreme and raising errors after just a few fetches. But if we were to increase that number when the first exception occurs we may have actually calculated some of them, they are, after all, separate tasks. Is there a way to retrieve these possibly completed tasks?

Why, yes, there is! More than one in fact, but let’s put a pin on that and check the [`gather` docs](https://docs.python.org/3.6/library/asyncio-task.html#asyncio.gather), notice there’s a `return_exceptions` argument that default to `False`:

> If `return_exceptions` is `True`, exceptions in the tasks are treated the same as successful results, and gathered in the result list; otherwise, the first raised exception will be **immediately** propagated to the returned future.

Note that this means `gather` **will not raise an exception anymore**, so we don’t need the `try..except` clause in our coroutine. We do, however, need to manually check the list of results to see if there were any errors.

In the following example I increased the number of allowed fetches to allow for some results to come back.

{% highlight python lineos %}
MAXIMUM_FETCHES = 500

# unchanged code

async def get_comments_of_top_stories(loop, session, limit, iteration):
    """Retrieve top stories in HN.
    """
    fetcher = URLFetcher()  # create a new fetcher for this task
    try:
        response = await fetcher.fetch(session, TOP_STORIES_URL)
    except BoomException as e:
        log.error("Error retrieving top stories: {}".format(e))
        # return instead of re-raising as it will go unnoticed
        return
    except Exception as e:  # catch generic exceptions
        log.error("Unexpected exception: {}".format(e))
        return

    tasks = [post_number_of_comments(
        loop, session, fetcher, post_id) for post_id in response[:limit]]

    # we're not using a try..except anymore as passing `return_exceptions`
    # as True will simply return the Exception object produced
    results = await asyncio.gather(*tasks, return_exceptions=True)

    # we can safely iterate the results
    for post_id, result in zip(response[:limit], results):
        # and manually check the types
        if isinstance(result, BoomException):
            log.error("Error retrieving comments for top stories: {}".format(
                result))
        elif isinstance(result, Exception):
            log.error("Unexpected exception: {}".format(result))
        else:
            log.info("Post {} has {} comments ({})".format(
                post_id, result, iteration))

    return fetcher.fetch_counter
{% endhighlight %}

```
[17:04:43] Calculating comments for top 5 stories. (1)
[17:04:43] Waiting for 5 seconds...
[17:04:47] Error retrieving comments for top stories: BOOM!
[17:04:47] Error retrieving comments for top stories: BOOM!
[17:04:47] Post 15051645 has 75 comments (1)
[17:04:47] Error retrieving comments for top stories: BOOM!
[17:04:47] Post 15052192 has 13 comments (1)
[17:04:47] > Calculating comments took 3.93 seconds and 511 fetches
[17:04:48] Calculating comments for top 5 stories. (2)
[17:04:48] Waiting for 5 seconds...
```

Look at that! We managed results for 2 out of the 5 top stories!

This particular example might require you to change the `MAXIMUM_FETCHES` depending on how popular the top stories in HN are at the point you’re running the script, I suggest increasing it to a high number allowing all tasks to complete and then lowering it to just below the fetch counter.

Let’s come back to that pin we put on the different ways to retrieve completed tasks:

## Gather vs wait (vs as_completed)

In the asyncio API there are two main functions for scheduling a set tasks at the same time, our familiar [`gather`](https://docs.python.org/3.5/library/asyncio-task.html#asyncio.gather) and [`wait`](https://docs.python.org/3.6/library/asyncio-task.html#asyncio.wait). The main differences between them are:

- `wait` returns a tuple of two sets of `Task` objects, **done** and **pending**, while `gather` returns the **results of those tasks**.
- `gather` returns the results **in order**, i.e. the first element of the returned list is the result of the `Task` object passed as a first parameter to it. In contrast, `wait` returns the objects **out of order** we need to manually keep track of which result corresponds to which `Task`.
- gather, by default, will return when an exception is raised by **any** of the tasks, **or** whenever **all tasks** are done if no exception is raised. As we’ve seen, if `return_exceptions` is `True` it will return when all tasks are “done” even if some raised an exception. Conversely `wait` has a specific parameter `return_when` that can be one of `FIRST_COMPLETED`, `FIRST_EXCEPTION` or `ALL_COMPLETED` allowing us finer control on when it returns.
- Finally, `wait` includes a `timeout` argument, `gather` does not have it but it’s possible to combine it with [`wait_for`](https://docs.python.org/3.6/library/asyncio-task.html#asyncio.wait_for) to mimic that behaviour.

Additionally, *asyncio* includes [`as_completed`](https://docs.python.org/3/library/asyncio-task.html#asyncio.as_completed) which returns an iterator of futures you can `await` as they are done. I cover both `wait` and `as_completed` in my other article [*Asyncio for the Working Python Developer*]({% post_url 2016-02-20-asyncio-for-the-working-python-developer %}) if you’re interested.

Why do I mention all this? Because we’re going to be needing it very soon.

## Cancelling tasks

It may not look like it, but we’re sending potentially thousands of requests to the HN API, more if we were to increase the number of top stories to get comments for, even more if any of the top stories happen to be controversial. We’re basically DDoS-ing the poor thing. Up until now it’s been fine with it but if we abuse it we may be getting some *429 Too Many Requests* or [*420 Enhance your calm*](https://dev.twitter.com/overview/api/response-codes).

So, ideally, as an exception is raised we should avoid making things worse for ourselves and cancel any scheduled tasks. Yeah, you heard me: *you can cancel the tasks*!

… but you need a handle on them, ideally we need handles as soon as an exception is raised. Now, as we mentioned before, `gather` returns the **results** of the tasks, but `wait` returns the `Task` objects in two tuples, *done* and *pending*.

Let’s give `wait` a go:

{% highlight python lineos %}
MAXIMUM_FETCHES = 500

# unchanged code

async def get_comments_of_top_stories(loop, session, limit, iteration):
    """Retrieve top stories in HN.
    """
    fetcher = URLFetcher()  # create a new fetcher for this task
    try:
        response = await fetcher.fetch(session, TOP_STORIES_URL)
    except BoomException as e:
        log.error("Error retrieving top stories: {}".format(e))
        # return instead of re-raising as it will go unnoticed
        return
    except Exception as e:  # catch generic exceptions
        log.error("Unexpected exception: {}".format(e))
        return

    tasks = [post_number_of_comments(
        loop, session, fetcher, post_id) for post_id in response[:limit]]

    # return on first exception to cancel any pending tasks
    done, pending = await asyncio.wait(tasks, return_when=FIRST_EXCEPTION)

    # cancel any pending tasks, the tuple could be empty so it's safe
    for pending_task in pending:
        pending_task.cancel()

    # process the done tasks
    for done_task in done:
        # one of the Tasks could raise an exception
        try:
            print("Post ??? has {} comments ({})".format(
                done_task.result(), iteration))
        except BoomException as e:
            print("Error retrieving comments for top stories: {}".format(e))

    return fetcher.fetch_counter
{% endhighlight %}

```
[17:20:04] Calculating comments for top 5 stories. (1)
[17:20:04] Waiting for 5 seconds...
Post ??? has 13 comments (1)
Error retrieving comments for top stories: BOOM!
[17:20:07] > Calculating comments took 3.79 seconds and 501 fetches
[17:20:09] Calculating comments for top 5 stories. (2)
[17:20:09] Waiting for 5 seconds...
Post ??? has 13 comments (2)
Post ??? has 82 comments (2)
Error retrieving comments for top stories: BOOM!
[17:20:13] > Calculating comments took 4.05 seconds and 501 fetches
[17:20:14] Calculating comments for top 5 stories. (3)
```

A few things to note on this example. First off the signature for `wait` is different, it accepts a **list of `Task` objects** so we don’t need to unpack the list as we were doing before with `gather`.

Secondly, we’re passing `return_when=FIRST_EXCEPTION`, specifically requesting it to return the two sets as soon as there’s an exception, at that point there will be pending tasks so we cancel them. If none are raised the second set will simply be empty. That’s why we managed to stop *immediately after hitting our upper limit for fetches* (500 in this example).

However, remember **done != successful**, if an exception was raised then one and only one `Task` object’s result is an exception, and calling `result` on it will raise it, so we need to be prepared to catch it.

Now for the caveats, notice the `???` in the post numbers? that’s because up until now we’ve been relying on `gather` returning the results **in order**, but we’ve lost that using `wait`.

What we need to do is keep track of which `Task` object corresponds to each post ID. Up until now we’ve been allowing `gather` and `wait` to create these `Task` objects from coroutine objects, but we can actually create the `Task` objects ourselves and pass the list to `wait`.

{% highlight python lineos %}
async def get_comments_of_top_stories(loop, session, limit, iteration):
    """Retrieve top stories in HN.
    """
    fetcher = URLFetcher()  # create a new fetcher for this task
    try:
        response = await fetcher.fetch(session, TOP_STORIES_URL)
    except BoomException as e:
        log.error("Error retrieving top stories: {}".format(e))
        # return instead of re-raising as it will go unnoticed
        return
    except Exception as e:  # catch generic exceptions
        log.error("Unexpected exception: {}".format(e))
        return

    tasks = {
        asyncio.ensure_future(
            post_number_of_comments(loop, session, fetcher, post_id)
        ): post_id for post_id in response[:limit]}

    # return on first exception to cancel any pending tasks
    done, pending = await asyncio.wait(
        tasks.keys(), return_when=FIRST_EXCEPTION)

    # if there are pending tasks is because there was an exception
    # cancel any pending tasks
    for pending_task in pending:
        pending_task.cancel()

    # process the done tasks
    for done_task in done:
        # if an exception is raised one of the Tasks will raise
        try:
            print("Post {} has {} comments ({})".format(
                tasks[done_task], done_task.result(), iteration))
        except BoomException as e:
            print("Error retrieving comments for top stories: {}".format(e))

    return fetcher.fetch_counter
{% endhighlight %}

```
[17:35:33] Calculating comments for top 5 stories. (1)
[17:35:33] Waiting for 5 seconds...
Error retrieving comments for top stories: BOOM!
Post 15051645 has 86 comments (1)
Post 15052192 has 16 comments (1)
[17:35:37] > Calculating comments took 4.06 seconds and 521 fetches
```

As you can see we can now get the post ID corresponding to our completed `Task` object from the set returned by `wait`.

Notice how we’re creating `Task` objects using `ensure_future` (we could’ve also used [`loop.create_task`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.AbstractEventLoop.create_task)) and then combining them using `wait`. Keeping references to our Task objects can be quite handy.

## Handling cancellation

Since we’re talking about cancelling tasks, the actual workflow is quite interesting. When `task.cancel()` is called a `CancelledError` exception is **sent to the coroutine**. The coroutine can actually catch that exception and act accordingly, it may even choose to ignore it.

In our case we wouldn’t want to ignore the cancellation entirely, but it might be useful to catch the exception to cancel all child tasks if we have any.

{% highlight python lineos %}
async def post_number_of_comments(loop, session, fetcher, post_id):
    """Retrieve data for current post and recursively for all comments.
    """
    url = URL_TEMPLATE.format(post_id)
    try:
        response = await fetcher.fetch(session, url)
    except BoomException as e:
        log.debug("Error retrieving post {}: {}".format(post_id, e))
        raise e

    # base case, there are no comments
    if response is None or 'kids' not in response:
        return 0

    # calculate this post's comments as number of comments
    number_of_comments = len(response['kids'])

    try:
        # create recursive tasks for all comments
        tasks = [asyncio.ensure_future(post_number_of_comments(
            loop, session, fetcher, kid_id)) for kid_id in response['kids']]

        # schedule the tasks and retrieve results
        try:
            results = await asyncio.gather(*tasks)
        except BoomException as e:
            log.debug("Error retrieving comments for top stories: {}".format(e))
            raise

        # reduce the descendents comments and add it to this post's
        number_of_comments += sum(results)
        log.debug('{:^6} > {} comments'.format(post_id, number_of_comments))

        return number_of_comments
    except asyncio.CancelledError:
        if tasks:
            log.info("Comments for post {} cancelled, cancelling {} child tasks".format(
                post_id, len(tasks)))
            for task in tasks:
                task.cancel()
        else:
            log.info("Comments for post {} cancelled".format(post_id))
        raise
{% endhighlight %}

```
[18:29:54] Calculating comments for top 5 stories. (1)
[18:29:54] Waiting for 5 seconds…
Post 13851386 has 1 comments (1)
Post 13851349 has 2 comments (1)
Error retrieving comments for top stories: BOOM!
[18:29:55] > Calculating comments took 1.76 seconds and 151 fetches
[18:29:55] Comments for post 13851706 cancelled, cancelling 3 child tasks
[18:29:55] Comments for post 13852103 cancelled, cancelling 1 child tasks
[18:29:55] Comments for post 13851611 cancelled, cancelling 2 child tasks
[... more messages like the above ...]
[18:29:59] Calculating comments for top 5 stories. (2)
[18:29:59] Waiting for 5 seconds…
```

The lesson here is that if there’s a chance that your coroutines can be cancelled remember you can catch the `CancelledError` and do any clean up that’s necessary. This also highlights the usefulness of storing references to any Tasks you schedule.

I find the solution quite elegant and Pythonic, exceptions are at the core of the language and still are even on a complicated feature like cancelling asynchronous tasks.

## Conclusion

And that’s all I have on *asyncio*, I hope these two articles have satisfied your hunger for coroutine knowledge and you’ve learned a few ideas to keep in mind while working with *asyncio*.

As you can see there’s quite a bit of housekeeping to perform, especially in situations where you want or need to deviate from *async/await* and start using different patterns. *Asyncio* has been subject of [detailed](http://lucumr.pocoo.org/2016/10/30/i-dont-understand-asyncio/) [analysis](https://vorpus.org/blog/some-thoughts-on-asynchronous-api-design-in-a-post-asyncawait-world/) from very clever people and sparked [third](https://curio.readthedocs.io/en/latest/) [party](https://trio.readthedocs.io/en/latest/) libraries that try a different approaches or aim to paliate our suffering.

Personally I think it is not without its flaws, but it’s had a tremendous impact in the community to embrace asynchronous programming. It’s really in its infancy and the core team is hard at work adding new features and polishing the API while the [ecosystem around it has flourished](https://github.com/timofurrer/awesome-asyncio) and it’s only getting better and better.

I believe *asyncio* has, without a doubt, pushed Python to the next level.
</div>
