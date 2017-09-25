---
layout: post
title:  "Asyncio Coroutine Patterns: Beyond await"
date:   2017-06-03 18:26:51 +0100
categories: blog
tags: python asyncio concurrency python3
image: asyncio-coroutine-patterns.jpg
---

*Asyncio*, the concurrent Python programmer's dream, write borderline synchronous code and let Python work out the rest, it's [`import antigravity`](http://xkcd.com/353/) all over again...

Except it isn't quite there yet, concurrent programming is hard and while coroutines allow us to avoid callback hell it can only get you so far, you still need to think about creating tasks, retrieving results and graceful handling of errors. Sad face.

Good news is all of that is possible in *asyncio*. Bad news is it's not always immediately obvious what wrong and how to fix it. Here are a few patterns I've noticed while working with *asyncio*.

![Medium](/assets/medium-50px.png) **[Continue reading *Asyncio Coroutine Patterns: Beyond await* on Medium](https://medium.com/python-pandemonium/asyncio-coroutine-patterns-beyond-await-a6121486656f)**