---
title: "Investigation into Docker"
cover: "2019-03-18/docker.png"
category: "development"
date: "2019-03-18"
tags:
  - docker
  - aws
  - devops
  - container
---

# Introduction to Docker

In my [previous post](https://foxnewsnetwork.github.io/teach-myself/investigation-into-kubernetes), I did a nascent investigation into kubernetes, what it is, and how to use it. From that investigation, I learned the following:

- kube is for scaling up containers across multiple physical machines

And came the conclusion that I needed put together my containers before I should get to figuring out their deployment strategies. So in this post, I will try to answer the following:

Q: What is docker?
Q: What is I docker-compose?

## What is Docker?

[Docker hub](https://hub.docker.com/), much like github, is a place where folks upload and share their containers (which can be thought of as computers with *very* specialized functionalities). Just like node.js projects on github can be run by any machine that has node.js installed, docker containers can be run by any machine that supports docker.

Before I go down the route of using docker to containerizes everything, though, I should conside some counterpoints. Starting with this [so-long docker article](https://www.linkedin.com/pulse/goodbye-docker-thanks-all-fish-maish-saidel-keesing/), here are some points I should investigate:

- [open container inititive](http://www.opencontainers.org/)

### TL;DR

- Docker as a run-time and a company is being superceded by podman and other native linux techs
- People at OCI are building container tools directly into Linux OS
- As an end user, I don't need to worry about underlying runtime and continue using docker containers and whatnot

## ECS and Fargate

For me, this means that the process through which I can get my graphing backend working is as follows:

- Create or compose local docker images that use graphviz and other tools
- Put together a lambda that will proxy to the images
- Ensure everything works on local docker / lambda
- Deploy to AWS [ECS](https://aws.amazon.com/ecs/) and [Fargate](https://aws.amazon.com/fargate/)
- Build slack bot to hit endpoint

![fargate diagram](2019-03-18/fargate.png)

## Docker Images

Eventual AWS deployment aside, I can get started on docker by running it locally. Let's get started with getting the [graphviz thing](https://hub.docker.com/r/omerio/graphviz-server) running locally. I'm honestly not too sure how to do everything, so let's just get started by doing and we can think / reason later

```zsh
mkdir sandbox-container
cd sandbox-container
docker pull omerio/graphviz-server
```

Something noteworthy things of running the above are:

- `docker/pull` does *NOT* actually pull down any files or folders
- Instead, running `docker images` lets us see what images are locally available:

```sh
➜  sandbox-container docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
omerio/graphviz-server   latest              d190726bbdb9        3 years ago         691MB
```

Then, running the container should be as simple as:

```zsh
docker run -d -p : omerio/graphviz-server
```

- `-d` is `--detach` as in run the process in the background
- `-p` is `--publish list` which means it'll publish the port we're operating on to console

In theory, after I've successfully started up a docker container, I should see the `docker container ls` command return something:

```zsh
➜  sandbox-container docker container ls
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```
Becomes:

```zsh
➜  teach-myself git:(master) ✗ docker container ls
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS              PORTS                     NAMES
387cfa3b5734        omerio/graphviz-server   "java -jar /opt/grap…"   9 seconds ago       Up 9 seconds        0.0.0.0:32768->4333/tcp   pensive_ellis
```

Then, using insomnia client, I am able to hit `http://localhost:32768/svg` with the `digraph G { a -> b -> c; a-> c; b -> d; c-> d; }` to generate the following digraph:

![generated digraph](2019-03-18/simple-digraph.png)

## Things for Next Time

- Figure out how to forward / configure ports
- Figure out where `Dockerfile` comes in
- Spin up and populate the `sandbox-container` project
- Go through the getting started docs
- Deploy services to AWS

# Appendix

## A1 - Getting Started Guide on Docker

[See here](https://docs.docker.com/docker-hub/)
