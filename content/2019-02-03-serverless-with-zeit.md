---
title: "Exploration into Serverless Backend with Zeit"
cover: "2019-02-03/zeit-co.png"
category: "development"
date: "2019-02-03"
tags:
  - serverless
  - lambda
  - investigation
  - research
  - backend
---

# Exploration into Serverless Platform with Zeit

Recently, I was reading Kent Dodd's blog post on [writing integration tests](https://blog.kentcdodds.com/write-tests-not-too-many-mostly-integration-5e8c7fff591c), and he casually means that Guerillmo Ranch, the creator of [socket.io](https://socket.io/), went on to create something called [zeit co](https://zeit.co/).

He mentions that this is some sort of serverless cloud competing platform.

To augment my ambition to build the UML slack bot, I'm in the market for a serverless cloud platform that is competitive with aws, kubernetes, azure, etc.,

This blog post will document my initial explorations into zeit and what that site offers (especially in contrast to running app-sync and lambda on aws)

## Zeit Now

Picking the bombastically generic name [Now](https://zeit.co/now) for their serverless platform, zeit offers what appears to be lambda-like service where they promise to run things like:

```javascript
module.exports = (req, res) => {
  res.send(doSomething(req))
}
```

>NOTE: it seems zeit does *NOT* offer things like AppSync, IAM, API Gateway, and the entire massive stack of shit that is a part of AWS

I follow the [installation guide](https://zeit.co/docs/v2/getting-started/installation/) and install this "now" thing via brew

```zsh
npm install -g now
```

as well as connect them to my github account

## Spinning up a New Project

Now apparently supports deployment with just github deployment; as such, I will follow the [monorepo guide](https://zeit.co/examples/monorepo/) and modify our existing [sandbox-serverless](https://github.com/foxnewsnetwork/sandbox-serverless) repo to resemble the [monorepo example](https://github.com/zeit/now-examples/tree/master/monorepo)

>Sidnote: it seems zeit now currently supports node, go, python, and (for some reason) php

In any case, after putting in a "hello-world"-esque `api/index.go` and `api/index.js` files, I move onto configuring my `now.json`

```json
{
  "version": 2,
  "name": "sandbox-serverless",
  "builds": [
    { "src": "api/go/*.go", "use": "@now/go" },
    { "src": "api/node/*.js", "use": "@now/node" }
  ],
  "routes": [
    { "src": "/api/(.*)", "dest": "/api/$1" }
  ]
}
```

Looking at this configuration, I would expect that I can make requests to `api/go/` to hit my `index.go` file.

>Protip: the `req` and `res` objects used as arguments to lambda correspond to [Client Request](https://nodejs.org/api/http.html#http_class_http_clientrequest) and [Server Response](https://nodejs.org/api/http.html#http_class_http_serverresponse) respectively

An exact dissertation on the configuration of `now.json` is found [here](https://zeit.co/docs/v2/deployments/configuration)

We deploy with the `now` command

## Testing our deployment

A deployment result looks like: 

![deployment results](2019-02-03/deploy.png)

Now, I will test my endpoint at `https://sandbox-serverless-jgwvcokcz.now.sh`

First up, I load up my [insomnia client](https://insomnia.rest/) against the go endpoint:

```
METHOD: GET
URL: https://sandbox-serverless-jgwvcokcz.now.sh/api/go/index.go
```

I have the following result (success)

![go test result](2019-02-03/go-result.png)

Now, the harder test is to attempt to load the nodejs svg image result

```
METHOD: GET
URL: https://sandbox-serverless-jgwvcokcz.now.sh/api/node/index.js
```

Since I expect this to be an SVG file, I can post this directly into the browser. Amazingly enough, it works!

![nodejs svg result](2019-02-03/node-result.png)

>Note: the `graphizo` thing label is unfortunate, so I'll probably have to generate my own images on lambda before I go live! @TODO for me

# Appendix

## A1 - TL;DR on Writing Tests, Not too many, and Mostly Integration

The gist of Kent Dodd's blog post on [writing integration tests](https://blog.kentcdodds.com/write-tests-not-too-many-mostly-integration-5e8c7fff591c) is summarized as:

- Unit tests
  - Good as training wheels to help the developer learn how to code
  - Cheap and write and run
  - Nearly completely useless for testing application
- Acceptance tests (aka application tests)
  - Tests *exactly* what the user will see, so provides application value
  - Expensive to write and slow to run
  - Brittle
- Integration tests
  - A good compromise between the unit and acceptance
  - Focus on writing these

Some side notes

- Coverage is a dumb metric to cover
- Coverage only matters in functional library code

## A2 - On Next.JS

While looking through the example docs, I come across this thing called "next.js"

![next js in tab bar](2019-02-03/nextjs.png)

After some research, it seems [next.js](https://nextjs.org/) is a react-based server-side rendering framework for building the so-called "progressive" web-app. This might be relevant for my actual work at SONY, though not so much for my side hobby projects.

In any case, I might wind up doing a blog post about it later on

## A3 - Graphizo

Although eventually, I would like to (at least partially) handle UML rendering in my own lambda, for now, I can use this URL based service called [graphizo](http://www.gravizo.com/) to get started.

Eventually, because I'd like to support

- animated flow-charts
- in-place editing
- image-annotating diagrams

that would require me to setup my own parse engine, I'd need to move off of graphizo

For funsies, here's an example of a graphizo rendered diagram:

<details>

<summary>

![Alt text](https://g.gravizo.com/svg?
  digraph G {
    aize ="4,4";
    main [shape=box];
    main -> parse [weight=8];
    parse -> execute;
    main -> init [style=dotted];
    main -> cleanup;
    execute -> { make_string; printf}
    init -> make_string;
    edge [color=red];
    main -> printf [style=bold,label="100 times"];
    make_string [label="make a string"];
    node [shape=box,style=filled,color=".7 .3 1.0"];
    execute -> compare;
  }
)

</summary>

```markdown
![Alt text](https://g.gravizo.com/svg?
  digraph G {
    aize ="4,4";
    main [shape=box];
    main -> parse [weight=8];
    parse -> execute;
    main -> init [style=dotted];
    main -> cleanup;
    execute -> { make_string; printf}
    init -> make_string;
    edge [color=red];
    main -> printf [style=bold,label="100 times"];
    make_string [label="make a string"];
    node [shape=box,style=filled,color=".7 .3 1.0"];
    execute -> compare;
  }
)
```

</details>
