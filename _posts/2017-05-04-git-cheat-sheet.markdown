---
published: false
title: Git cheat sheet
layout: post
tags: [git]
categories: []
---

# Show all releases with dates

```
git log --tags --simplify-by-decoration --pretty="format:%ai %d %an" | grep "tag: v" | grep "2017-"
```
