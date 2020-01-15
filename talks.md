---
layout: default
title: Talks
description: "Talks I've given"
nav: true
---

# Import as an anti-pattern: Demystifying Dependency Injection in Python

> Dependency Injection in Python is commonly seen as over-engineering, but I think this is a myth. DI is simple and powerful and can yield great benefits to the overall quality of your code.

I gave this talk at PyCon UK 2019 in Cardiff ([slides](https://speakerdeck.com/yeray/import-as-an-antipattern-demystifying-dependency-injection-in-python)):

<div class="video-container">
<iframe src="https://www.youtube.com/embed/qkGxy4c64Jg" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

## Abstract

Dependency Injection (DI) is a technique quite common in other programming languages that has never quite fitted into the Python programmer's mindset. Usually disregarded as over-engineering and complex, most of its alleged flaws have little to do with the underlying concepts but more with implementation details of large scale frameworks that obscure its core principles.

In my experience DI is simple and powerful and can greatly improve our design making our code is easier to test, extend and maintain.

In this talk I will start with a brief standard example and challenge common idioms that introduce coupling and reduce maintainability and extensibility. I will then introduce concepts like interfaces, inversion of control and DI and apply them showcasing how the design and overall quality of the code improves.

Along the way I will demonstrate how recent additions to Python and its tooling can help when applying these techniques and also address the common misconception among Python developers suggesting large complex frameworks are required to apply DI.

I will then share some examples of Dependency Injection in well known libraries and some hands-on stories where I have benefited from it in my day-to-day work. And finally I will provide some guidelines on identifying situations where DI shines and where its application might be less obvious or should be delayed until further domain knowledge is introduced.

With this talk I hope to demystify the Dependency Injection in the Python ecosystem as well as introduce and clarify important design concepts for a beginner to intermediate audience.
