---
layout: post
title: "Workload Capture & Replay"
tags: Work Programming
permalink: /blog/workload-capture-replay/
---
When customers hit issues in production, it can be an effort to locally reproduce them, especially when external sources are involved. Reproducing issues is useful not just to figure out the root cause, but also to verify the fix and add a regression test. The newly introduced workload capture & replay tooling records a Materialize instance's state as well as recent queries and ingestion rates, then replays them in a Docker Compose environment with synthetic data. In this blog post I’ll show how it works and talk about some of the challenges and future work.

Read the rest of the blog post over on the [Materialize blog](https://materialize.com/blog/materialize-workload-capture-replay/).
