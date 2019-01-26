---
title: "Hello Gatsby"
cover: "2019-01-26-hello-gatsby.png"
category: "development"
date: "2019-26-01"
tags:
    - gatsby
    - blog
    - setup
    - howto
---

# Hello Gatsby

I've finally decided to maintain an actual presentable blog instead of just using markdown files, gists, etc., to capture documentation.

This is simply the "hello world" post so don't expect much content here.

Next time (as in this weekend), I plan to discuss the following two topics

- What is the quantum eraser effect?
  - What does it have to do with time travel?
  - Does *any* information travel backward in time?
- Explorations in to the slack API
  - How to setup a slack bot project

## Deployment

After clearing out `contents/*.md` and other such unused things, I attempted to build and deploy with:

```sh
yarn run build:gh
```

However, I was rudely greeted with the following graphQL field error:

![graphql error](2019-26-01/deploy-error.png)

This suggests that something is wrong with my `date` field... hmmm, what could it be?
