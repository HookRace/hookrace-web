---
layout: post
title: "Speeding up Materialize CI"
tags: Work Programming
permalink: /blog/speeding-up-materialize-ci/
---
In the [previous post](https://materialize.com/blog/qa-process-overview/) I talked about how we test Materialize. This time I’ll describe how I significantly sped up our Continuous Integration (CI) Test pipeline in July, especially for pull requests that require a build and full test run. The goal is to make developers more productive by reducing the time waiting for CI to complete.

We always kept CI runtime in mind, but it still slowly crept up over the years through adding tests, the code itself growing larger, as well as hundreds of minor cuts adding up.

Read the rest of the blog post over on the [Materialize blog](https://materialize.com/blog/speeding-up-materialize-ci/).
