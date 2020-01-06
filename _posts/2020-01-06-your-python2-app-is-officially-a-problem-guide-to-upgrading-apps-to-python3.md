---
layout: post
title: "It's 2020, your Python 2.7 app is officially a problem — A guide to upgrading apps to Python 3"
date: 2020-01-06 09:51:19 +0000
categories: python
tags: python
image: python-clock.png
caption: <a href="https://pythonclock.org">pythonclock.org</a>
applause: true
---

<div markdown="1" class="sticky">
* TOC
{:toc}
<div class="feedback"><applause-button color="#424242" multiclap="true" /></div>
</div>

<div markdown="1" id="text">

And just like that, [Python 2.7 has reached end-of-life](https://pythonclock.org/), and your Python 2.7 app has officially become a problem.

Where do you even start? Everything will need to change!

I've been there and it's not great. It's not just the changes to the code, there's many other factors to keep in mind while upgrading and a lot of potential problems caused by subtle and not so subtle differences in Python itself and your dependencies.

I had to go through this process recently for a 140k lines of code app first and a couple times more for smaller services and, as one does, I worked out a system which I now present to you in guide form.

You will not find an exhaustive list of syntax differences between Python 2 and 3 in this guide. This is a pragmatic, hands-on workflow to upgrade your app to Python 3 that worked for me, hopefully it will help you too.

<!--more-->

## Step 0: Setting the stage

Be warned, this is a fairly long process depending on the size of the codebase. If the code is being actively worked on you want to merge as many steps compatible with Python 2 as possible to avoid having one big `python3` branch that you'll have to merge changes onto, explain changes to your team and deploy.

I suggest having a quick meeting with your team to get on the same page in terms of Python 3 and how it will impact your code. Leaving comments with a common prefix, something like `# PY3: ...`, in places where the idioms must or could change between versions was quite helpful for me. That way you can quickly search for that string and apply the necessary changes later on.

Ready? Let's get to it!

## Step 1: It all starts with tests

The larger project I upgraded didn't have what you'd call an exhaustive test coverage but there were _some_ tests in virtually all the important bits. They were mostly unit tests which were less ideal for this task but I was glad there were at least some.

You should have an idea of how well covered sections of the code are, if you don't stop and add [`coverage.py`](https://coverage.readthedocs.io/) right now, I'll wait.

You'll probably find some important places that lack tests it's worth stopping and adding tests for them. Even if the logic is simple, write a single test, not only it will improve coverage but the tests will also cover the imports in that module which will catch any potential import errors you may introduce while upgrading.

As you analyse the existing tests make a note of the types of data that are consumed and any text, binary data or file processing the application is doing and where. Typically these are the boundaries of the application, places like the database, interfaces to other services or a filesystem.

This is important because the Python 2, as you may know, treats text and binary data differently than Python 3. As you upgrade a lot of errors will come from usage of `str` objects as binary data, whereas Python 3 will want to use `bytes` objects. Also take note of how the tests deal with these interactions, typically Python 2 tests will use some string literals or `StringIO` objects, annotate where they're used, it will be useful later.

Finally, write down manual tests to make sure that, as you upgrade, things are still working correctly. Obviously if you can automate them even better, but in my experience setting up a working end-to-end test suite is non-trivial and can take you a fairly long time. These can be as simple as firing up the development server and going through some critical path writing down the steps.

This step will take some time, but it's most definitely worth it, spend whatever time you need on it. Hopefully you don't need to convince anyone about the value of adding tests but there's potential for some frustration as you start reading and testing code that's been hidden away for a while.

Keep at it, you'll be happy you did later.

## Step 2: Gotta resolve those dependencies

Everyone handles dependencies differently so this step is slightly opinionated. Since we're dealing with an officially old application I expect you'll be relying on the good ol' `pip freeze > requiremements.txt` method.

However, the final resolved dependencies **will be different between Python 2 and 3** so it's important to establish the high level dependencies of the application and have a reliable process to resolve them as we upgrade. This, again, is a worthwhile change to add even if you are not upgrading to Python 3.

### Identifying top level dependencies

First I used [`pipdeptree`](https://github.com/naiquevin/pipdeptree) to identify the top-level dependencies. The README includes a [recipe](https://github.com/naiquevin/pipdeptree#using-pipdeptree-to-write-requirementstxt-file) to do exactly that. Once you've got your list separate them in "main", "testing" and "development" dependencies and write them into three different `*.in` files. Keep the versions pinned for now, or at least limited to minor versions where applicable.

This might surprise you but keep in mind the goal is to port the application. Upgrading the dependencies is a different problem that can be tackled once we have a stable Python 3 codebase.

At this point you'll likely realize some of the libraries can be quite outdated. Chances your key dependencies will have support for Python 3 since it's been around for a while now, but you'll have to choose how much you can or want to upgrade. I suggest looking at the change logs to check the minimal version you can upgrade to minimizing the amount of code changes you have to make. Again, our goal is to have our app working in Python 3 with as little problems as possible.

In the worst case some them will not have support for Python 3. If you find yourself in that situation I suggest having a read through [Anthony Shaw's guide to upgrading a dependency](https://medium.com/python-pandemonium/oh-no-this-package-is-python-2-only-8e6316f9a02), but here is a quick rundown:

- Have a look at potential forks, it's likely others have had the same problem and implemented the necessary changes, at least partially.
- If you do find a fork or have to implement the changes yourself PLEASE, PLEASE, PLEASE, open a pull request to the upstream project.
- If the project is abandoned and you're interested in maintaining it you can open a [PEP 541](https://www.python.org/dev/peps/pep-0541/) request in [PyPI support](https://github.com/pypa/pypi-support).

### Resolving dependencies

Once we have these three files for `main`, `testing` and `dev` dependencies, I used [`pip-tools`](https://github.com/jazzband/pip-tools) to transform these three files onto a fully resolved `requirements.txt` file to feed onto `pip`. There are [other tools for this task](https://hynek.me/articles/python-app-deps-2018/), but I find `pip-tools` is by far the less invasive and appropriate for our task.

The end goal of this step is to have _reproducible_ installations not only for production but for CI and development. This will help minimize errors introduced by mismatches in dependencies across environments.

`pip-tools` itself is composed of two different CLI tools, personally I only used the `pip-compile` command to create three pinned requirement files, `main.txt`, `test.txt` and `dev.txt` from three similarly named `*.in` files we created in the previous step.

The process is as follows:

1. `pip-compile --output-file main.txt main.in`, this will create `main.txt` with only the core pinned dependencies.
2. `pip-compile --output-file test.txt main.txt test.in`, notice how we're starting with the _pinned_ `main.txt` dependencies first and then adding the top-level `test.in` dependencies. This will ensure the main dependencies stay pinned and condition the test dependencies to them.
3. `pip-compile --output-file test.txt dev.in`, similarly, we start with the pinned test dependencies, which also include the core ones, and add the top level development dependencies.

Typically you'd want to script this process onto a single command to create all three files. Remember to update any `pip install -r` commands to point to the appropriate file.

Once you've got your final requirements files quickly check against the old `requirements.txt` for any missing dependencies, create a new Python 2.7 virtual environment, install the dependencies from the `dev.txt` file and run the tests.

If any dependencies are missing there will be quick import errors and should be easy to fix, simply add the top level dependency in the appropriate file and run the script to generate the three files again.

Again, this step is fairly opinionated on the tooling and workflow, but it is _crucial_ as you upgrade to Python 3. Make sure you have this or another solution to resolve your dependencies reliably before moving forward.

This step and the previous one can be safely merged onto your Python 2.7 codebase, in fact it's probably advisable to start introducing them to reduce the amount of changes your team will have to go through. It will also help debug any potential problems before blaming everything on the Python 3 upgrade (trust me, they'll be a lot of that).

## Step 3: Upgrading the actual code

It's now time to update the code itself!

Good news is Python 3 has been around long enough that mature code upgrading tools are available to ease this process.

I chose [`python-future`](http://python-future.org/), specifically the [`futurize` tool](http://python-future.org/automatic_conversion.html), which separates the process in three stages:

1. "Safe" fixes, which performs fairly straightforward changes using Python's `2to3` and some custom changes without making the code require the library itself.
2. Wrapper fixes, which performs more complex changes using wrappers from the library, for example installing standard library aliases.
3. Unicode literals, which will add `from __future__ import unicode_literals`, effectively assuming all string literals are unicode unless explicitly marked as `bytes`.

I performed the first two steps in separate commits, they're quite safe but they will produce quite a few changes. Make sure you run the tests after applying them to be on the safe side.

In the second step `python-future` will introduce some imports to patch the Python 2.7 standard library so calls to it take the same form as Python 3. The imports look like the following:

```
from future import standard_library
standard_library.install_aliases()
```

This is will *not* make your linter happy, it might be worth disabling that rule temporarily, then again it will be a good reminder to clean them up later. Note this means you will have to add `python-future` to your `main.in` requirements file.

The third step will probably break things, in my experience most codebases don't use string literals as binaries except in very specific cases. A good example that bit me were tests that manually set a user's password to a binary literal, which, after applying this last step caused to fail. Another example were tests that were using `StringIO` as _binary_ containers, `python-future` will _not_ change those to the `BytesIO` equivalent, so trying to populate those directly with string literals will fail.

At this point the code will work in Python 2.7 and Python 3! This sound counter intuitive, but it was invaluable to me to be able to go back to the Python 2.7 virtual environment and check if the errors were really caused by the Python 3 upgrade. Performing a one way migration, particularly in low test coverage situations, would be quite risky.

### Finally, Python 3!

It's time to create your first Python 3 virtual environment.

First off you need to decide which version of Python 3 you want to run. You may want to upgrade to the latest and greatest but you may want to hold off depending on the state of your libraries. Aim for a version that all your core libraries support, that being said most libraries that support a fairly recent version of Python 3 will work on newer versions even if the support is not official.

If you're using Linux or Mac I highly recommend [`pyenv`](https://github.com/yyuu/pyenv). It will greatly simplify moving back and forth between the two versions of Python and different virtual environments.

At this point we need to resolve the dependencies for Python 3, so before we install anything onto our new virtual environment we'll need to install `pip-tools` and run the script to resolve the dependencies.

You'll notice the end resulting files will be different which is to be expected since a lot of libraries have conditional subdependencies for helpers in Python 2.7 or are able to use more modern versions of their dependencies as a result of the upgrade. Do note this means we've performed _some_ dependency upgrades, but not on our top level dependencies. This _may_ introduce some errors but usually libraries do a good job on make this transparent for the user.

After resolving our dependencies in Python 3 we can now install them as usual and things should just work, right?

Spoiler alert: they won't.

## Step 4: While True: Test, fail, fix

Tests will fail _hard_ (probably).

To avoid getting overwhelmed I recommend using the "stop after first failure" option or the excellent [stepwise](https://docs.pytest.org/en/latest/cache.html#stepwise) function in `pytest`. This will allow you to iterate quickly on the test suite without running already passing tests.

The types of errors you'll get will be mostly:

### Import errors

These will come up quite quickly, typically before the actual tests start. They're caused by a missing dependency in our new requirement files that was likely not in the original requirements file but was installed as a subdependency and used directly. I had annoying instances where the imports where *not* at the module level, which caused errors later on because the tests did not exercise the code that imported.

The best way to tackle these is to switch back to your working Python 2.7 virtual environment and run `pipdeptree` once more searching for the missing package. Add the package at the specific version to your `main.in` file and run the script to resolve the dependencies again.

### Type and attribute errors

As you progress onto the actual tests you'll likely get a _lot_ of `AttributeError: 'str' has no attribute 'decode'` as the unicode string literals are tried to be decoded in different parts of the system.

Unfortunately, the only thing you can really do is remove these calls to `decode`, a pretty tedious job let me tell you, but they should be easy to spot. I'd discourage doing a search and replace though. There's likely going to be situations where the tests really want to pass a `bytes` literal, particularly when mocking at some boundary of the application like the file system or a database row with specific binary column. The notes we took in step 1 around this fact should come in handy right about now.

You're also bound to see some very amusing strings like `'b"Some message"'`, caused by string formatting `bytes` objects. These are quite hard to spot and might slip through the testing phase.

### Dependency API errors

Some errors might be caused by breaking changes in the API of dependencies. That includes the standard library, unfortunately. `future` will patch a lot functionality for you but there's slight changes in behaviours between Python 2 and 3 in some places.

These types of errors take quite a bit of work as you'll have to go through change logs or worse the actual dependency's diffs between the old and new versions.

A particular example in one of the upgrades I performed was a library changing a parameter from a ISO 8601 string to a datetime object. I recommend using a debugger just before the call that's causing the error and stepping through the dependency's code as it can be quite hard to realize what the problem is only from diffs.

### Errors during manual tests

Fixing automated tests will take a _while_, but you _will_ get through it, and eventually you'll see a lovely sequence of green dots. It's time to do some manual testing and, you guessed it, they will fail (probably).

Of course this depends heavily on your application, its tests and where they fall in the unit vs. end-to-end spectrum, but in my experience the set of manual test we wrote in our first step will come in really handy.

The errors will likely be similar to the ones you've already fixed, they will simply not be covered by the automated tests.

An example of one of my upgrades was a change in the standard library's `pickle` module that caused Python 3 to crash when attempting to read cached objects. That was a fun use of 3 hours.

Annoying and slow as this process is you'll be glad you're finding these errors now rather than in production.

## Step 5: It's merging time!

Amazing, all tests, automated and manual are looking good! Congratulations!

We still need to tie up some loose ends though, here's a quick list of what I had to do at this point:

- Communicate with the team your intention to merge your branch.
- Gauge if it's worth waiting for features in development by other member of the team to be merged first or if it's worth porting those changes to Python 3 after your merge.
- Merge `master` onto your branch and run tests again. If you feel lucky try rebasing, but merging tends to be simpler in my experience.
- Go through any points you or your colleagues have commented as `# PY3`.
- Change any Dockerfiles as required and/or let the more ops savvy members of your team know the changes that need to happen, they'll love it, trust me.
- Update the developer documentation regarding the setup of their new Python 3 environment.
- Tag the previous stable Python 2.7 version.
- Update the setup.py and bump the version.
- Merge your branch.
- Officially communicate to the team the changes are now in place.

## Step 6: Put it in prod

It's all fun and games until you put it in production...

Make no mistake, it's stressful, depending on your continuous delivery policy or lack thereof.

Hopefully you'll have a series of environments where changes to can be tested before going to production. Inevitably something will go wrong so make sure deploy at a sensible time of day and keep an eye on the logs and error reports.

Fair warning, there will be a _lot_ of blaming Python 3, and you will get pinged quite a bit more about errors than you used to, but things will quiet down after a while.

Take this time between fixing errors to clean up some of the Python 2/3 compatibility elements from `future` as you won't be needing those anymore. A simple search and replace with an empty string worked quite well for me.

## Congratulations, you are now in Python 3!

So there you have it, your app is now in Python 3.

As I mentioned above, this process is not fun and takes a while depending on the size of your application. The monolith I had to upgrade took me about a week to get all tests to pass and another to put it in production. However, everyone was quite happy and grateful in the end.

I hope this guide will help you have an easier time upgrading your app so you can finally start using `async`/`await`.

That's the whole reason you're upgrading, right?

</div>
