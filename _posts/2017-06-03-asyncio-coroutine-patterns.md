---
layout: post
title:  "Asyncio Coroutine Patterns: Beyond await"
date:   2017-06-03 18:26:51 +0100
categories: python asyncio
tags: python asyncio python3
image: asyncio-coroutine-patterns.jpg
---

<div markdown="1" class="sticky">
* TOC
{:toc}
</div>

<div markdown="1" id="text">
*Asyncio*, the concurrent Python programmer's dream, write borderline synchronous code and let Python work out the rest, it's [`import antigravity`](http://xkcd.com/353/) all over again...

Except it isn't quite there yet, concurrent programming is hard and while coroutines allow us to avoid callback hell it can only get you so far, you still need to think about creating tasks, retrieving results and graceful handling of errors. Sad face.

Good news is all of that is possible in *asyncio*. Bad news is it's not always immediately obvious what wrong and how to fix it. Here are a few patterns I've noticed while working with *asyncio*.

<!--more-->

Really quick before we start, I’ll be using the lovely [`aiohttp`](https://aiohttp.readthedocs.io/) library to make asynchronous HTTP requests and the [Hacker News API](https://github.com/HackerNews/API) because it’s simple and a well-known site that follows a familiar use case. I’ll also be using the `async/await` syntax introduced in Python 3.5, following feedback from my [previous article]({% post_url 2016-02-20-asyncio-for-the-working-python-developer %}), I will assume the reader is familiar with the concepts described there. And finally all examples are available in [this article’s GitHub repo](https://github.com/yeraydiazdiaz/asyncio-coroutine-patterns).

Right, let’s get started!

## Recursive coroutines

Creating and scheduling tasks is trivial in *asyncio*. The API includes several methods in the [`AbstractEventLoop`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.AbstractEventLoop) class and well as functions in the library for that purpose. But usually you want to combine the results of those tasks and process them in some way, recursion is a perfect example of this pattern and also shows off the simplicity of coroutines versus other means of concurrency.

A common use case for *asyncio* is to create some kind of webcrawler. Imagine we’re just too busy to check HackerNews, or maybe you just like a good ol’ flamewar, so you want to implement a system that retrieves the number of comments for a particular HN post and if it’s over a threshold notify you. You google for a bit and find the HN API docs, just what we need, however you notice this in its docs:

> Want to know the total number of comments on an article? Traverse the tree and count.

Challenge accepted!

{% highlight python lineos %}
"""
A recursive function solves a problem by simplifying the input until
we arrive at a base trivial case and then combining the results up the stack.
Assume we want to calculate the number of comments of a particular post in
Hacker News by recursively aggregating the number of descendents.
"""

import asyncio
import argparse
import logging
from urllib.parse import urlparse, parse_qs
from datetime import datetime

import aiohttp
import async_timeout


LOGGER_FORMAT = '%(asctime)s %(message)s'
URL_TEMPLATE = "https://hacker-news.firebaseio.com/v0/item/{}.json"
FETCH_TIMEOUT = 10

parser = argparse.ArgumentParser(
    description='Calculate the comments of a Hacker News post.')
parser.add_argument('--id', type=int, default=8863,
                    help='ID of the post in HN, defaults to 8863')
parser.add_argument('--url', type=str, help='URL of a post in HN')
parser.add_argument('--verbose', action='store_true', help='Detailed output')


logging.basicConfig(format=LOGGER_FORMAT, datefmt='[%H:%M:%S]')
log = logging.getLogger()
log.setLevel(logging.INFO)

fetch_counter = 0


async def fetch(session, url):
    """Fetch a URL using aiohttp returning parsed JSON response.
    As suggested by the aiohttp docs we reuse the session.
    """
    global fetch_counter
    with async_timeout.timeout(FETCH_TIMEOUT):
        fetch_counter += 1
        async with session.get(url) as response:
            return await response.json()


async def post_number_of_comments(loop, session, post_id):
    """Retrieve data for current post and recursively for all comments.
    """
    url = URL_TEMPLATE.format(post_id)
    now = datetime.now()
    response = await fetch(session, url)
    log.debug('{:^6} > Fetching of {} took {} seconds'.format(
        post_id, url, (datetime.now() - now).total_seconds()))

    if 'kids' not in response:  # base case, there are no comments
        return 0

    # calculate this post's comments as number of comments
    number_of_comments = len(response['kids'])

    # create recursive tasks for all comments
    log.debug('{:^6} > Fetching {} child posts'.format(
        post_id, number_of_comments))
    tasks = [post_number_of_comments(
        loop, session, kid_id) for kid_id in response['kids']]

    # schedule the tasks and retrieve results
    results = await asyncio.gather(*tasks)

    # reduce the descendents comments and add it to this post's
    number_of_comments += sum(results)
    log.debug('{:^6} > {} comments'.format(post_id, number_of_comments))

    return number_of_comments


def id_from_HN_url(url):
    """Returns the value of the `id` query arg of a URL if present, or None.
    """
    parse_result = urlparse(url)
    try:
        return parse_qs(parse_result.query)['id'][0]
    except (KeyError, IndexError):
        return None


async def main(loop, post_id):
    """Async entry point coroutine.
    """
    now = datetime.now()
    async with aiohttp.ClientSession(loop=loop) as session:
        now = datetime.now()
        comments = await post_number_of_comments(loop, session, post_id)
        log.info(
            '> Calculating comments took {:.2f} seconds and {} fetches'.format(
                (datetime.now() - now).total_seconds(), fetch_counter))

    return comments


if __name__ == '__main__':
    args = parser.parse_args()
    if args.verbose:
        log.setLevel(logging.DEBUG)

    post_id = id_from_HN_url(args.url) if args.url else args.id

    loop = asyncio.get_event_loop()
    comments = loop.run_until_complete(main(loop, post_id))
    log.info("-- Post {} has {} comments".format(post_id, comments))

    loop.close()
{% endhighlight %}

Feel free to try it out with `--verbose` flag for a more detailed output

```
[14:47:32] > Calculating comments took 2.23 seconds and 73 fetches
[14:47:32] -- Post 8863 has 72 comments
```

Let’s skip the boilerplate and go directly to the recursive coroutine, notice it reads almost exactly as it would in synchronous code:

{% highlight python lineos %}
async def post_number_of_comments(loop, session, post_id):
    """Retrieve data for current post and recursively for all comments.
    """
    url = URL_TEMPLATE.format(post_id)
    now = datetime.now()
    response = await fetch(session, url)
    log.debug('{:^6} > Fetching of {} took {} seconds'.format(
        post_id, url, (datetime.now() - now).total_seconds()))

    if 'kids' not in response:  # base case, there are no comments
        return 0

    # calculate this post's comments as number of comments
    number_of_comments = len(response['kids'])

    # create recursive tasks for all comments
    log.debug('{:^6} > Fetching {} child posts'.format(
        post_id, number_of_comments))
    tasks = [post_number_of_comments(
        loop, session, kid_id) for kid_id in response['kids']]

    # schedule the tasks and retrieve results
    results = await asyncio.gather(*tasks)

    # reduce the descendents comments and add it to this post's
    number_of_comments += sum(results)
    log.debug('{:^6} > {} comments'.format(post_id, number_of_comments))

    return number_of_comments
{% endhighlight %}

1. First retrieve the post’s JSON.
2. Recurse for each of the descendants.
3. Eventually arrive at a base case and return zero when the post has no replies.
4. On returning from the base case add the replies to the current post to that of its descendants and return.

This is a perfect example of what Brett Slatkin describes as [*fan-in and fan-out*](https://www.youtube.com/watch?v=CWmq-jtkemY), we *fan-out* to retrieve the data for the descendants and *fan-in* reducing the data retrieved to calculate the number of comments.

Asyncio’s API has a couple of ways to perform this fan-out operation, here I’m using `gather` which effectively waits until all coroutines are done and returns a list of their results.

Note how using a coroutine also fits nicely with recursion since at any one point there are any number coroutines awaiting their responses on the gather call and resuming execution after the I/O operation is done and allowing us to express a fairly complicated behaviour in a single nice and readable coroutine.

“Too easy”, you say? Ok, let’s turn in up a notch.

## Fire and forget

Imagine you want to send yourself an email with any posts with a number of comments above a certain threshold, and you’d like to do that as we traverse the tree of posts. We could simply add an `if` statement at the end of our recursive function to do so:

{% highlight python lineos %}
async def post_number_of_comments(loop, session, post_id):
    """Retrieve data for current post and recursively for all comments.
    """
    url = URL_TEMPLATE.format(post_id)
    response = await fetch(session, url)

    if 'kids' not in response:  # base case, there are no comments
        return 0

    # calculate this post's comments as number of comments
    number_of_comments = len(response['kids'])

    # create recursive tasks for all comments
    tasks = [post_number_of_comments(
        loop, session, kid_id) for kid_id in response['kids']]

    # schedule the tasks and retrieve results
    results = await asyncio.gather(*tasks)

    # reduce the descendents comments and add it to this post's
    number_of_comments += sum(results)
    log.debug('{:^6} > {} comments'.format(post_id, number_of_comments))

    # Log if number of comments is over a threshold
    if number_of_comments > MIN_COMMENTS:
        await log_post(response)

    return number_of_comments


async def log_post(post):
    """Simulate logging of a post.
    """
    await asyncio.sleep(random() * 3)
    log.info("Post logged")
{% endhighlight %}

Yes, I’m using asyncio.sleep, this is the last time, I promise.

```
[09:41:02] Post logged
[09:41:04] Post logged
[09:41:06] Post logged
[09:41:06] > Calculating comments took 6.35 seconds and 73 fetches
[09:41:06] -- Post 8863 has 72 comments
```

That is quite slower than before! The reason is that, as we discussed before, `await` suspends the coroutine until the future is done, but since we don’t need the result of the logging there’s no real reason to do so.

We need to *fire and forget* our coroutine, but since we can’t `await` it, we need another way to schedule the execution of the coroutine without waiting on it. A quick look at the asyncio API yields [`ensure_future`](https://docs.python.org/3/library/asyncio-task.html#asyncio.ensure_future) (TODO: change to create_task), which will schedule a coroutine to be run, wrap it in a Task object and return it. Remember that, once scheduled, the event loop will yield control to our coroutine at some point in the future when another coroutine *awaits*. Cool, let’s swap `await log_post` with it then:

{% highlight python lineos %}
async def post_number_of_comments(loop, session, post_id):
    """Retrieve data for current post and recursively for all comments.
    """
    url = URL_TEMPLATE.format(post_id)
    response = await fetch(session, url)

    if 'kids' not in response:  # base case, there are no comments
        return 0

    # calculate this post's comments as number of comments
    number_of_comments = len(response['kids'])

    # create recursive tasks for all comments
    tasks = [post_number_of_comments(
        loop, session, kid_id) for kid_id in response['kids']]

    # schedule the tasks and retrieve results
    results = await asyncio.gather(*tasks)

    # reduce the descendents comments and add it to this post's
    number_of_comments += sum(results)
    log.debug('{:^6} > {} comments'.format(post_id, number_of_comments))

    # Log if number of comments is over a threshold
    if number_of_comments > MIN_COMMENTS:
        asyncio.ensure_future(log_post(response))

    return number_of_comments
{% endhighlight %}

```
[09:42:57] > Calculating comments took 1.69 seconds and 73 fetches
[09:42:57] -- Post 8863 has 72 comments
[09:42:57] Task was destroyed but it is pending!
task: <Task pending coro=<log_post() done, defined at 02_fire_and_forget.py:82> wait_for=<Future pending cb=[<TaskWakeupMethWrapper object at 0x1109197f8>()]>>
[09:42:57] Task was destroyed but it is pending!
task: <Task pending coro=<log_post() done, defined at 02_fire_and_forget.py:82> wait_for=<Future pending cb=[<TaskWakeupMethWrapper object at 0x110919948>()]>>
[09:42:57] Task was destroyed but it is pending!
task: <Task pending coro=<log_post() done, defined at 02_fire_and_forget.py:82> wait_for=<Future pending cb=[<TaskWakeupMethWrapper object at 0x110919978>()]>>
```

Ah, the dreaded `Task was destroyed but it is pending!`, haunting users of *asyncio* around the globe. Good news is we’re back to the times we were getting before (1.69 secs), the bad news is *asyncio* is not liking out fire-and-forget.

The problem is that we’re forcibly closing the loop right after the `post_number_of_comments` coroutine returns, leaving our `log_post` tasks no time to complete.

We have two options, we either let the loop [run forever](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.AbstractEventLoop.run_forever) and manually abort the script, or use the [`all_tasks`](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task.all_tasks) Task class method to find any pending tasks and await the once we’re done calculating comments. Let’s try that out with a quick change after our call to `post_number_of_comments`:

{% highlight python lineos %}
if __name__ == '__main__':
    args = parser.parse_args()
    if args.verbose:
        log.setLevel(logging.DEBUG)

    post_id = id_from_HN_url(args.url) if args.url else args.id

    loop = asyncio.get_event_loop()
    comments = loop.run_until_complete(main(loop, post_id))
    log.info("-- Post {} has {} comments".format(post_id, comments))

    pending_tasks = [
        task for task in asyncio.Task.all_tasks() if not task.done()]
    loop.run_until_complete(asyncio.gather(*pending_tasks))

    loop.close()
{% endhighlight %}

```
[09:47:29] > Calculating comments took 1.72 seconds and 73 fetches
[09:47:29] — Post 8863 has 72 comments
[09:47:30] Post logged
[09:47:31] Post logged
[09:47:32] Post logged
```

Now we’re ensuring the logging tasks complete. Relying on `all_tasks` works fine in situations where we have a good idea of what tasks are due to be executed in our event loop, but on more complex examples there could be any number of tasks pending whose origin might not even be in our code.

Another approach is to clean up after ourselves by registering any and all coroutines *we’ve* scheduled and allow the pending ones to be executed once we’re done calculating comments. As we know `ensure_future` returns a Task object we can use to register our low priority tasks. Let’s simply define a `task_registry` list and store the futures in it:

{% highlight python lineos %}
async def post_number_of_comments(loop, session, post_id):
    """Retrieve data for current post and recursively for all comments.
    """
    url = URL_TEMPLATE.format(post_id)
    response = await fetch(session, url)

    if 'kids' not in response:  # base case, there are no comments
        return 0

    # calculate this post's comments as number of comments
    number_of_comments = len(response['kids'])

    # create recursive tasks for all comments
    tasks = [post_number_of_comments(
        loop, session, kid_id) for kid_id in response['kids']]

    # schedule the tasks and retrieve results
    results = await asyncio.gather(*tasks)

    # reduce the descendents comments and add it to this post's
    number_of_comments += sum(results)
    log.debug('{:^6} > {} comments'.format(post_id, number_of_comments))

    # Log if number of comments is over a threshold
    if number_of_comments > MIN_COMMENTS:
        # Add the future to the registry
        task_registry.append(asyncio.ensure_future(log_post(response)))

    return number_of_comments

# (... ommitted code ...) #

if __name__ == '__main__':
    args = parser.parse_args()
    if args.verbose:
        log.setLevel(logging.DEBUG)

    post_id = id_from_HN_url(args.url) if args.url else args.id
    task_registry = []  # define our task registry

    loop = asyncio.get_event_loop()
    comments = loop.run_until_complete(main(loop, post_id))
    log.info("-- Post {} has {} comments".format(post_id, comments))

    pending_tasks = [
        task for task in task_registry if not task.done()]
    loop.run_until_complete(asyncio.gather(*pending_tasks))

    loop.close()
{% endhighlight %}

```
[09:53:46] > Calculating comments took 1.68 seconds and 73 fetches
[09:53:46] — Post 8863 has 72 comments
[09:53:46] Post logged
[09:53:48] Post logged
[09:53:49] Post logged
```

The lesson here is *asyncio* should not be treated as a distributed job queue such as [*Celery*](http://celeryproject.org/), it all runs under a single thread and the event loop needs to be managed accordingly allowing it time to complete the tasks.

Which leads to another common pattern:

## Periodic coroutines

Continuing with our HN example, and since we’ve done such a great job already, we’ve decided it’s absolutely crucial to calculate the number of comments of stories in HN as they appear and while they’re at the top 5 new entries.

A quick look at the HN API reveals an endpoint that returns the 500 latest stories, perfect, so we could simply poll that endpoint to retrieve new stories and calculate the number of comments on them every, say, five seconds.

Right, so since we are going to poll periodically we can just use an infinite `while` loop, `await` the polling task and `sleep` for the period of time necessary. I’ve added a few minor changes so as to retrieve the top stories instead of a specific post’s URL:

{% highlight python lineos %}
"""
An example of periodically scheduling coroutines using an infinite loop of
awaiting and sleeping.
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

fetch_counter = 0


async def fetch(session, url):
    """Fetch a URL using aiohttp returning parsed JSON response.
    As suggested by the aiohttp docs we reuse the session.
    """
    global fetch_counter
    with async_timeout.timeout(FETCH_TIMEOUT):
        fetch_counter += 1
        async with session.get(url) as response:
            return await response.json()


async def post_number_of_comments(loop, session, post_id):
    """Retrieve data for current post and recursively for all comments.
    """
    url = URL_TEMPLATE.format(post_id)
    response = await fetch(session, url)

    if 'kids' not in response:  # base case, there are no comments
        return 0

    # calculate this post's comments as number of comments
    number_of_comments = len(response['kids'])

    # create recursive tasks for all comments
    tasks = [post_number_of_comments(
        loop, session, kid_id) for kid_id in response['kids']]

    # schedule the tasks and retrieve results
    results = await asyncio.gather(*tasks)

    # reduce the descendents comments and add it to this post's
    number_of_comments += sum(results)
    log.debug('{:^6} > {} comments'.format(post_id, number_of_comments))

    return number_of_comments


async def get_comments_of_top_stories(loop, session, limit, iteration):
    """Retrieve top stories in HN.
    """
    response = await fetch(session, TOP_STORIES_URL)
    tasks = [post_number_of_comments(
        loop, session, post_id) for post_id in response[:limit]]
    results = await asyncio.gather(*tasks)
    for post_id, num_comments in zip(response[:limit], results):
        log.info("Post {} has {} comments ({})".format(
            post_id, num_comments, iteration))


async def poll_top_stories_for_comments(loop, session, period, limit):
    """Periodically poll for new stories and retrieve number of comments.
    """
    global fetch_counter
    iteration = 1
    while True:
        now = datetime.now()
        log.info("Calculating comments for top {} stories. ({})".format(
            limit, iteration))
        await get_comments_of_top_stories(loop, session, limit, iteration)

        log.info(
            '> Calculating comments took {:.2f} seconds and {} fetches'.format(
                (datetime.now() - now).total_seconds(), fetch_counter))
        log.info("Waiting for {} seconds...".format(period))
        iteration += 1
        fetch_counter = 0
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

```
[10:14:03] Calculating comments for top 5 stories. (1)
[10:14:06] Post 13848196 has 31 comments (1)
[10:14:06] Post 13849430 has 37 comments (1)
[10:14:06] Post 13849037 has 15 comments (1)
[10:14:06] Post 13845337 has 128 comments (1)
[10:14:06] Post 13847465 has 27 comments (1)
[10:14:06] > Calculating comments took 2.96 seconds and 244 fetches
[10:14:06] Waiting for 5 seconds…
[10:14:11] Calculating comments for top 5 stories. (2)
[10:14:14] Post 13848196 has 31 comments (2)
[10:14:14] Post 13849430 has 37 comments (2)
[10:14:14] Post 13849037 has 15 comments (2)
[10:14:14] Post 13845337 has 128 comments (2)
[10:14:14] Post 13847465 has 27 comments (2)
[10:14:14] > Calculating comments took 3.04 seconds and 244 fetches
[10:14:14] Waiting for 5 seconds…
```

Nice, but there’s a slight problem: if you notice the timestamps it’s not strictly running the task every 5 seconds, it runs it 5 seconds **after `get_comments_of_top_stories` finishes**. Again a consequence of using `await` and blocking until we get our results back. Which might not be a problem except in cases where the task takes longer than five seconds. Also, it feels kind of wrong to use `run_until_complete` on a coroutine designed to be infinite.

Good news is we’re experts on ensure_future now, instead of awaiting we can just slap that guy in and …

{% highlight python lineos %}
async def poll_top_stories_for_comments(loop, session, period, limit):
    """Periodically poll for new stories and retrieve number of comments.
    """
    global fetch_counter
    iteration = 1
    while True:
        now = datetime.now()
        log.info("Calculating comments for top {} stories. ({})".format(
            limit, iteration))
        asyncio.ensure_future(
            get_comments_of_top_stories(loop, session, limit, iteration))

        log.info(
            '> Calculating comments took {:.2f} seconds and {} fetches'.format(
                (datetime.now() - now).total_seconds(), fetch_counter))
        log.info("Waiting for {} seconds...".format(period))
        iteration += 1
        fetch_counter = 0
        await asyncio.sleep(period)
{% endhighlight %}

```
[10:55:40] Calculating comments for top 5 stories. (1)
[10:55:40] > Calculating comments took 0.00 seconds and 0 fetches
[10:55:40] Waiting for 5 seconds…
[10:55:43] Post 13848196 has 32 comments (1)
[10:55:43] Post 13849430 has 48 comments (1)
[10:55:43] Post 13849037 has 16 comments (1)
[10:55:43] Post 13845337 has 129 comments (1)
[10:55:43] Post 13847465 has 29 comments (1)
[10:55:45] Calculating comments for top 5 stories. (2)
[10:55:45] > Calculating comments took 0.00 seconds and 260 fetches
[10:55:45] Waiting for 5 seconds…
[10:55:48] Post 13848196 has 32 comments (2)
[10:55:48] Post 13849430 has 48 comments (2)
[10:55:48] Post 13849037 has 16 comments (2)
[10:55:48] Post 13845337 has 129 comments (2)
[10:55:48] Post 13847465 has 29 comments (2)
```

O… k… good news is the timestamps are spaced exactly by five seconds, but what’s with the zero seconds and no fetches? And then the next iteration took zero seconds and 260 fetches?

This is one of the consequences of moving away from `await`, since we’re not blocking anymore the coroutine simply moves to the next line which prints the zero seconds and, the first time, zero fetches message. These are fairly trivial problems since we can live without the messages, but what if we needed the result of the task?

Then, my friend, we’d need to resort to… **callbacks** (*shudder*)

I know, I know, the whole point of coroutines is to not use callbacks, but this is why the article’s dramatic subtitle is “Beyond await”. We’re not in *await-land* anymore, we’re adventuring into manually scheduling tasks to meet our use case. Do you have what it takes? (hint: it’s not that bad)

As we discussed before `ensure_future` returns a `Future` object to which we can add a callback to using `add_done_callback`.

Before we do that, and in order to have proper counting of fetches we’re going to have to encapsulate our fetch coroutine into a class called `URLFetcher`, then make an instance per task to have proper fetch counting, also removing that global variable that was bugging me anyway:

{% highlight python lineos %}
"""
An example of periodically scheduling coroutines using an infinite loop of
scheduling a task using ensure_future and sleeping.
Using a callback to the future returned by ensure_future we can now
correctly output statistics on the elapsed time and fetches using the new
URLFetcher class. But at a cost in readability.
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

```
[12:23:40] Calculating comments for top 5 stories. (1)
[12:23:40] Waiting for 5 seconds...
[12:23:43] Post 13848196 has 38 comments (1)
[12:23:43] Post 13849430 has 72 comments (1)
[12:23:43] Post 13849037 has 19 comments (1)
[12:23:43] Post 13848283 has 64 comments (1)
[12:23:43] Post 13847465 has 34 comments (1)
[12:23:43] > Calculating comments took 3.17 seconds and 233 fetches
[12:23:45] Calculating comments for top 5 stories. (2)
[12:23:45] Waiting for 5 seconds...
[12:23:47] Post 13848196 has 38 comments (2)
[12:23:47] Post 13849430 has 72 comments (2)
[12:23:47] Post 13849037 has 19 comments (2)
[12:23:47] Post 13848283 has 64 comments (2)
[12:23:47] Post 13847465 has 34 comments (2)
[12:23:47] > Calculating comments took 2.47 seconds and 233 fetches
[12:23:50] Calculating comments for top 5 stories. (3)
[12:23:50] Waiting for 5 seconds...
```

Right, that’s better, but let’s focus on the callback section:

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

Notice the `callback` function [needs to accept a single argument](https://docs.python.org/3/library/asyncio-task.html#asyncio.Future.add_done_callback), the future it’s assigned to. We’re also returning the fetch count from the `URLFetcher` instance as a result of `get_comments_of_top_stories` and retrieving it as the result of the future.

See? I told you it wasn’t that bad, but it’s no `await` that’s for sure.

While we’re on the subject of callbacks, on your inevitable journeys into the asyncio API docs you’re likely to find a couple of methods in `AbstractBaseLoop` by the name of `call_later` and its cousin `call_at` which sound like something useful to implement periodic coroutines. And you would be right, it can be used, we just need make a couple of changes:

{% highlight python lineos %}
# ... ommitted code ...

async def get_comments_of_top_stories(loop, limit, iteration):
    """Retrieve top stories in HN.
    """
    async with aiohttp.ClientSession(loop=loop) as session:
        fetcher = URLFetcher()  # create a new fetcher for this task
        response = await fetcher.fetch(session, TOP_STORIES_URL)
        tasks = [post_number_of_comments(
            loop, session, fetcher, post_id) for post_id in response[:limit]]
        results = await asyncio.gather(*tasks)
        for post_id, num_comments in zip(response[:limit], results):
            log.info("Post {} has {} comments ({})".format(
                post_id, num_comments, iteration))
        return fetcher.fetch_counter  # return the fetch count

def poll_top_stories_for_comments(loop, period, limit, iteration=0):
    """Periodic function that schedules get_comments_of_top_stories.
    """
    log.info("Calculating comments for top {} stories ({})".format(
        limit, iteration))

    future = asyncio.ensure_future(
        get_comments_of_top_stories(loop, limit, iteration))

    now = datetime.now()

    def callback(fut):
        fetch_count = fut.result()
        log.info(
            '> Calculating comments took {:.2f} seconds and {} fetches'.format(
                (datetime.now() - now).total_seconds(), fetch_count))

    future.add_done_callback(callback)

    log.info("Waiting for {} seconds...".format(period))

    iteration += 1
    loop.call_later(
        period,
        partial(  # or call_at(loop.time() + period)
            poll_top_stories_for_comments,
            loop, period, limit, iteration
        )
    )


if __name__ == '__main__':
    args = parser.parse_args()
    if args.verbose:
        log.setLevel(logging.DEBUG)

    loop = asyncio.get_event_loop()

    # we don't `run_until_complete` anymore, we simply call the function
    poll_top_stories_for_comments(loop, args.period, args.limit)

    # and run the loop forever
    loop.run_forever()

    loop.close()
{% endhighlight %}

Resulting in a similar output as before. Notice a few changes though:

- We no longer have a `main` entry point coroutine, instead…
- `poll_top_stories_for_comments` is now a standard function that schedules tasks. Due to this we moved the creation of the `aiohttp.ClientSession` to `get_comments_of_top_stories` which is our first coroutine. Note this is simply to work around `aiohttp’s` requirement of `async with` when creating as session. The important bit is…
- We moved away from the original pattern of infinite-loop-and-sleeping to using `ensure_future` to schedule the coroutine and then the function scheduling itself for later execution. One could argue this approach is [less explicit](https://www.python.org/dev/peps/pep-0020/#id3). Finally…
- Since on every iteration `poll_top_stories_for_comments` uses the loop to schedule itself we obviously have to use to `run_forever` for the loop to always be running.

OK, that’s all fine and dandy, but what if, God forbid, our connection broke in the middle of a task? What would happen to our lovely system? Let’s simulate that by introducing a raised exception after a number of URL fetches:

{% highlight python lineos %}
MAXIMUM_FETCHES = 5

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
                raise Exception('BOOM!')

            async with session.get(url) as response:
                return await response.json()
{% endhighlight %}

```
[12:51:00] Calculating comments for top 5 stories. (1)
[12:51:00] Waiting for 5 seconds…
[12:51:01] Exception in callback poll_top_stories_for_comments.<locals>.callback(<Task finishe…ion(‘BOOM!’,)>) at 05_periodic_coroutines.py:121
handle: <Handle poll_top_stories_for_comments.<locals>.callback(<Task finishe…ion(‘BOOM!’,)>) at 05_periodic_coroutines.py:121>
Traceback (most recent call last):
 File “/Users/yeray/.pyenv/versions/3.6.0/lib/python3.6/asyncio/events.py”, line 126, in _run
 self._callback(*self._args)
 File “05_periodic_coroutines.py”, line 122, in callback
 fetch_count = fut.result()
 File “05_periodic_coroutines.py”, line 100, in get_comments_of_top_stories
 results = await asyncio.gather(*tasks)
 File “05_periodic_coroutines.py”, line 69, in post_number_of_comments
 response = await fetcher.fetch(session, url)
 File “05_periodic_coroutines.py”, line 58, in fetch
 raise Exception(‘BOOM!’)
Exception: BOOM!
[12:51:05] Calculating comments for top 5 stories. (2)
[12:51:05] Waiting for 5 seconds…
```

Not exactly graceful, is it?

What to do? Head over to the next part of this series where I explore the options we have for error handling and other issues: [*Asyncio Coroutine Patterns: Errors and Cancellation*]({% post_url 2017-08-29-asyncio-coroutine-patterns-errors-and-cancellation %}).
</div>
