---
layout: post
title:  "Asyncio Coroutine Patterns: Errors and Cancellation"
date:   2017-08-29 14:08:51 +0100
categories: blog
tags: python asyncio concurrency python3
image: asyncio-coroutine-patterns-errors-and-cancellation.jpg
---

This is the second part of a two part series on coroutine patterns in *asyncio*, to fully benefit from this article please read the first installment: [*Asyncio Coroutine Patterns: Beyond await*]()

In the first part of this series we concluded that *asyncio* is awesome, coroutines are awesome and our code is awesome. But sometimes the outside world is not as awesome and we have to deal with it.

Now, for this second part of the series, I'll run over the options *asyncio* gives us to handle errors when using these patterns and cancelling tasks so as to make our asynchronous systems robust and performant.

If you are really new to *asyncio* I recommend having a read through my very first article on the subject, [*Asyncio for the Working Python Developer*](https://hackernoon.com/asyncio-for-the-working-python-developer-5c468e6e2e8e), before diving into this series.

I will be continuing with the examples using the Hacker News API and also be using the async/await syntax introduced Python 3.5+ and all example code is available in the Github repo for this series.

![Medium](/assets/medium-50px.png) **[Continue reading *Asyncio Coroutine Patterns: Errors and Cancellation* on Medium](https://medium.com/@yeraydiazdiaz/asyncio-coroutine-patterns-errors-and-cancellation-3bb422e961ff)**