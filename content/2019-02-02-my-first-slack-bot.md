---
title: "My First Slack Bot"
cover: "2019-02-02/slack-bot.png"
category: "development"
date: "2019-02-02"
tags:
  - slack
  - bot
  - setup
  - howto
---

# Putting Together a Slack Bot

Here, I document and go over the steps it took me to setup a slack bot.

The ultimate vision is eventually get this slack bot to process UML and drawing languages and return diagrams directly into slack, then charge companies some amount of money for this service.


## Going through documentation

I start out by read through [slack bot docs](https://api.slack.com/bot-users#getting-started)

I click the Create App button and name my bot - in this case "diagram"

![create app screenshot](2019-02-02/create-app.png)

## App Design

Here's what I want this app to do:

- When given a command like the following:

```text
\diagram flow 

st=>start: Start:>http://www.google.com[blank]
e=>end:>http://www.google.com
op1=>operation: My Operation
sub1=>subroutine: My Subroutine
cond=>condition: Yes
or No?:>http://www.google.com
io=>inputoutput: catch something...
para=>parallel: parallel tasks

st->op1->cond
cond(yes)->io->e
cond(no)->para
para(path1, bottom)->sub1(right)->op1
para(path2, top)->op1

```

- I would like this to lead to an image like they have on [flowchart.js](http://flowchart.js.org/)

```
https://some-cdn-host.com/hash-val.png
```

![rendered output](2019-02-02/flow-chart.png)

## Creating New Commands

Under the [MyApps page](https://api.slack.com/apps/AFVRCMG72/slash-commands?) on slack, I go into `Features > Slash Commands` to create my desired command

The API presented here seems quite intuitive, and demands that I will need:

- command
  - `\diagram`
- request url
  - I'll need to setup my lambda back-end here
  - Q: What actually does the request to my back-end look like?
  - Q: What will my response need to be?
- description
  - Draws a graph!
- usage hints
  - Show above hint

![creating new slack commands](2019-02-02/create-new-cmd.png)

But before I can proceed, I 

## The Internals of a Slack Command Request

According to the [slack API docs](https://api.slack.com/slash-commands#app_command_handling), we have the following example:

- `/weather 94070` leads to the following response

```
token=gIkuvaNzQIHg97ATvDxqgjtO
&team_id=T0001
&team_domain=example
&enterprise_id=E0001
&enterprise_name=Globular%20Construct%20Inc
&channel_id=C2147483705
&channel_name=test
&user_id=U2147483697
&user_name=Steve
&command=/weather
&text=94070
&response_url=https://hooks.slack.com/commands/1234/5678
&trigger_id=13345224609.738474920.8088930838d88f008e0
```

With header `Content-Type: application/x-www-form-urlencoded` 

For us, the fields `text` will store the directives to draw our UML file, and possibly `team_id` / `enterprise_id` for when we decide to track usage and costs

>Note: if our response takes longer than 3s, we'll have to use the `response_url` field and send back `200` right away

## A Slack Command Response

In slack, we have 3000 ms (3 seconds) to make a resposne before we are considered timeout. In that time, we must respond with the following

- `status: 200`
- `content-type: application/json` 

Here's the type of expected response (see [composing messages docs](https://api.slack.com/docs/messages#composing_messages) for more details)

>Protip: a lot of this requires experimentation on the [slack message builder](https://api.slack.com/docs/messages/builder?msg=%7B%22text%22%3A%22We%20should%20be%20concerned%20if%20the%20variable%20value%20for%20%60radioactive%60%20is%20%60true%60.%22%7D)

```typescript
type SlackResponse = {
  response_type: 'ephemeral' | 'in_channel',
  text?: string,
  attachments: Array<Attachment>
};

type Attachment = {
  text?: string
};
```

>Sidenote: we could also send back plain text, but that's not applicable to us

Here's an example response (taken from the [slack messages article](https://api.slack.com/docs/message-attachments))

```json
{
  "response_type": "in_channel",
  "text": "clc no",
  "attachments": [
    {
      "fallback": "Required plain-text summary of the attachment.",
      "color": "#2eb886",
      "pretext": "Optional text that appears above the attachment block",
      "author_name": "Bobby Tables",
      "author_link": "http://flickr.com/bobby/",
      "author_icon": "http://flickr.com/icons/bobby.jpg",
      "title": "Slack API Documentation",
      "title_link": "https://api.slack.com/",
      "text": "Optional text that appears within the attachment",
      "fields": [
        {
          "title": "Priority",
          "value": "High",
          "short": false
        }
      ],
      "image_url": "https://avatars0.githubusercontent.com/u/9582?s=460&v=4",
      "thumb_url": "https://avatars3.githubusercontent.com/u/1481354?s=70&v=4",
      "footer": "Slack API",
      "footer_icon": "https://avatars2.githubusercontent.com/u/6510388?s=70&v=4",
      "ts": 123456789
    }
  ]
}
```

>Note: Slack messages appear to be tailor-made for sharing articles and links! So not everything is possible

As it turns out, slack is smart enough to auto-process images:

So something as simple as:
```json
{
  "attachments": [
    {
      "fallback": "Uh-oh",
      "image_url": "https://i.imgur.com/5yDgqz7.jpg",
      "actions": [
        {
          "type": "button",
          "text": "Edit",
          "url": "https://flights.example.com/book/r123456",
          "style": "primary"
        }
      ]
    }
  ]
}
```

Will generate the [following](https://api.slack.com/docs/messages/builder?msg=%7B%22attachments%22%3A%5B%7B%22fallback%22%3A%22Uh-oh%22%2C%22image_url%22%3A%22https%3A%2F%2Fi.imgur.com%2F5yDgqz7.jpg%22%2C%22actions%22%3A%5B%7B%22type%22%3A%22button%22%2C%22text%22%3A%22Edit%22%2C%22url%22%3A%22https%3A%2F%2Fflights.example.com%2Fbook%2Fr123456%22%2C%22style%22%3A%22primary%22%7D%5D%7D%5D%7D)

![generated message](2019-02-02/message.png)

## Next time

In part 2 of this blog, we will go over:

- Setting up lambda
- connecting with graphizo
