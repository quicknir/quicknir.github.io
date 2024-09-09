---
layout: post
title: "Setting up a C++ (and python) environment on Linux with micromamba (and VsCode)"
---

I've been shilling micromamba a fair bit on both C++ and python servers that I'm a part of. Kind of become
"that" guy. I've been promising people for a while I'll give a brief intro on it, and especially, helping
people get started with a C++ development environment. Because it's definitely not trivial.

This is that intro. The earlier sections cover setting up a micromamba environment with everything you need,
and is relevant to both languages. Then I'll talk about building C++ code and setting up VsCode - this part is
just for C++ because in Python this stuff will just work (of course).

The only prerequisite is a sane version of Linux. WSL is also fine (where I'm testing this in fact). MacOS
is also fine, but the command to  (WSL is fine - where this is
being tested in fact) and basic command line
familiarity. if you care about the VsCode setup, then have it installed and have
basic familiarity (or be ready to Google) as I won't cover that.

## Install Micromamba
Throughout this post I'm going to work entirely in one directory so that at the end, if you're not pleased, you
can easily delete everything.

```bash
cd # Switch to home directory
mkdir asp && cd asp
```

{% highlight bash %}
cd # Switch to home directory
mkdir asp && cd asp
{% endhighlight %}

The command given is for Linux x86_64....

Next, add the following lines to your .bashrc (if you use zsh - then add it to your .zshrc).

{% highlight bash linenos %}
export MAMBA_ROOT_PREFIX=~/asp/micromamba  # optional, defaults to ~/micromamba
eval "$(~/asp/bin/micromamba shell hook -s posix)"
# micromamba activate default_env
{% endhighlight %}

Notice the last line is commented - we'll come back to that. Go ahead and source your bashrc.

## Install an environment

Next, run

{% highlight bash linenos %}
micromamba create -f default_env.yaml
{% endhighlight %}

You'll be shown a list of everything being installed and prompted - hit y. Installing should just take a few
seconds.