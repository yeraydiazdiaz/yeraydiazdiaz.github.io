---
layout: post
title:  "What the mock? — A cheatsheet for mocking in Python"
date:   2018-05-09 10:48:19 +0100
categories: blog
tags: unit testing python mocking tools
image: what-the-mock-a-cheatsheet-for-mocking-in-python.jpg
---

It's just a fact of life, as code grows eventually you will need to start adding mocks to your test suite. What started as a cute little two class project is now talking to external services and you cannot test it comfortably anymore.

That's why Python ships with unittest.mock, a powerful part of the standard library for stubbing dependencies and mocking side effects.

However, unittest.mock is not particularly intuitive.

I've found myself many times wondering why my go-to recipe does not work for a particular case, so I've put together this cheatsheet to help myself and others get mocks working quickly.

<div markdown="1" class="medium">
**[Continue reading *What the mock? — A cheatsheet for mocking in Python* on Medium](https://medium.com/@yeraydiazdiaz/what-the-mock-cheatsheet-mocking-in-python-6a71db997832)**
</div>