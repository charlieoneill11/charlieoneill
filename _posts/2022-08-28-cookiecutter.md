---
title: "Reproducible data science in the real-world"
description: "Putting the science back in data science"
layout: post
toc: true
comments: true
hide: false
search_exclude: true
categories: [data science, python, analysis, statistics, software development]
---

Data science often swings between two extremes: ensuring that the output of the work is polished, or focusing on the code which generates that output. Unfortunately, this imbalanced approach leads to problems on both sides. Only thinking about the end product can reduce correctness of code and reproducibility, and cause generate significant technical debt. Conversely, pedantically writing code like a software developer can reduce the agility of a data science team, and prevent them from quickly iterating on experiments to find "good-enough" solutions. Here, I aim to outline the process used by Macuject's data science team to strike an appropriate balance between the two extremes.