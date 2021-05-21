---
layout: post
title: "Setting up Sublime Text 4 for Python"
date: 2021-05-21 09:33:19 +0000
categories: python
tags: python
image: st4_python.jpg
short_description: How to setup Sublime Text 4 for Python using Python LSP Server
keywords: "python sublimetext4"
---

<div markdown="1" class="sticky">
* TOC
{:toc}
</div>

<div markdown="1" id="text">

[Sublime Text](https://www.sublimetext.com) has had its first major upgrade since 2017 with [Sublime Text 4](https://www.sublimetext.com/blog/articles/sublime-text-4).

Sublime Text is not an IDE nor it pretends to be, but its powerful plugin system has allowed the community to come up with clever ways to have some IDE capabilities.

I've been using ST4 in its beta releases exclusively for some time and arrived at a setup that I enjoy, hopefully you will to.

I will update this post with any developments in the tools and setup.

<!--more-->

**Disclaimer**: This setup will **not** make ST4 behave like PyCharm or even like VSCode. When you choose ST4 you are sacrificing some IDE features for speed and performance.

## The tools

I've used Sublime Text for Python since I bought my first ST3 license in 2017. At the time the state-of-the-art way of working in Python was to use [Anaconda](https://github.com/DamnWidget/anaconda) (not to be confused with [Anaconda, the Python distribution](https://anaconda.org/)). Anaconda still works in ST4 but it's unfortunately lacking some maintenance and its approach has been superseded by language servers.

Sublime Text has official support for language servers through its [LSP project](https://lsp.sublimetext.io/) allowing plugins to call into different language servers and render results in ST4, and, of course, Python is no exception.

There are two main language servers available for ST4: [Pyright](https://github.com/sublimelsp/LSP-pyright) and [Python LSP Server](https://github.com/python-lsp/python-lsp-server/). Pyright is straight forward to set up, simply follow the instructions in the README, but it will only yield static typing information.

I will focus this article on setting up Python LSP Server with a set of common plugins.

## The setup

1. Install `LSP` using the package manager:

2. In your project's virtual environment (you are using a virtual environment, right?)

```
pip install python-language-server[all] pyls-black mypy-ls pyls-isort
```

And any other PyLSP plugins you see fit, these are the ones I use most.

{:start="3"}
3. Go to the Sublime Text preferences, Package Settings, LSP, Settings, to open the general settings for LSP where we will configure some default options for PyLSP and its plugins.
![Sublime Text 4 LSP Preferences](/assets/st4/LSP_settings.png "Sublime Text 4 LSP Preferences")

4. Add the following:

```jsonc
{
    "clients": {
        "pylsp": {
            "enabled": false,  // we will enable PyLSP at the project level
            "selector": "source.python",
            "settings": {
                "pylsp.plugins.pyflakes.enabled": false, // enabled by default, use flake8
                "pylsp.plugins.pycodestyle.enabled": false, // enabled by default, use flake8
                "pylsp.plugins.flake8.enabled": true,  // flake8 is included in pyls
                "pylsp.configurationSources": [
                  "flake8",   // discover flake8 config in ~/.config/flake8, setup.cfg, tox.ini and flake8.cfg
                ],
                "pylsp.plugins.jedi_rename.enabled": true,  // included in pyls
                // File formatter, invoke via LSP: Format file
                "pylsp.plugins.autopep8.enabled": false,  // enabled by default, use black
                "pylsp.plugins.black.enabled": true,  // from pyls-black
                "pylsp.plugins.mypy_ls.enabled": true,  // from mypy-ls
            },
        },
    },
}

```

{:start="5"}
5. Save your Sublime Text Project if you haven't yet and edit its configuration as follows:

```jsonc
{
    // This section should be present already
    "folders":
    [
        {
            "path": "<ABSOLUTE_PATH_TO_YOUR_PROJECT>",
        }
    ],
    // Add this whole section if not present or just the LSP settings
    "settings": {
        "LSP": {
            "pylsp": {
                "enabled": true,
                "command": [
                    "<ABSOLUTE_PATH_TO_YOUR_VENV>/bin/pylsp",
                ],
                "settings": {
                    "pylsp.plugins.flake8.executable": "<ABSOLUTE_PATH_TO_YOUR_VENV>/bin/flake8",
                },
            },
        },
    },
}
```

Make sure you replace `<ABSOLUTE_PATH_TO_YOUR_VENV>` with the absolute path to your virtual environment.

{:start="6"}
6. Test it out in your project:

- Hovering over a symbol should render information about it with links to its definition and references:
![Hover over symbol](/assets/st4/st4_hover.gif "Hover over symbol")

- You should see Flake8 and mypy errors as you type by hovering over the warning and error squiggly lines or openin the LSP diagnostics panel with `LSP: Toggle Diagnostics Panel`:
![Flake8 and mypy errors](/assets/st4/st4_mypy_flake8.gif "Flake8 and mypy errors")

- Invoke `LSP: Format File` to format with Black and isort:
![Black and isort on format](/assets/st4/st4_format_file.gif "Black and isort on format")

- Invoke `LSP: Rename` to rename a symbol using Jedi
![Jedi rename symbol](/assets/st4/st4_rename.gif "Jedi rename symbol")

## Troubleshooting

If you find some of the plugins are not working or ST4 is showing an error about PyLSP crashing, you may want to add the following lines to the `command` list in the project's config:

```jsonc
"command": [
    "<ABSOLUTE_PATH_TO_YOUR_VENV>/bin/pylsp",
    "--log-file",
    "<ABSOLUTE_PATH_TO_A_LOG_FILE>",
    "--verbose"
],
```

The server should restart on each change to the project's configuration file, but you may need to restart ST4 (luckily it's lightning fast). You should see the output of PyLSP in that log file which will help you debug any problems.

If you find any issues with this setup let me know in [this blog's repo](https://github.com/yeraydiazdiaz/yeray.dev/issues).

## Extras

You may also want to install the following Sublime Text packages:

- [MagicPython](https://github.com/MagicStack/MagicPython)

## Acknowledgements

I would like to thank the Spyder IDE team for taking the time to fork and maintain [Python LSP Server](https://github.com/python-lsp/python-lsp-server/) as well as to all the maintainers of the different plugins for PyLSP for adapting to the fork and releasing new versions promptly.

And, of course, the Sublime HQ team for making an awesome text editor.

</div>
