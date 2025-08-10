---
layout: post
author: Al Calzone
title: "AI for logs? Just GZIP it!"
summary: |-
  Or: <i>How to make Z-Wave JS logs more approachable with AI</i>
excerpt: |-
  Or: <i>How to make Z-Wave JS logs more approachable with AI</i>
# image: chameleon-coding.jpeg
# feature: chameleon-coding.jpeg
comments: true
---

AI is all the hype these days, and I've been using my fair share of it. Mostly to assist me with writing code when I'd rather get a coffee, but also to reduce some friction for users when contributing to Z-Wave JS. Even though the logos of many AI companies look eerily similar to a [certain part of human anatomy](https://velvetshark.com/ai-company-logos-that-look-like-buttholes), the results their tools produce are (often) far from what normally comes out of there.

As such it is no surprise that AI came up during the recent meetup where we did Z-Wave range tests [in a real world scenario]({% link _posts/2025-08-01-zwave-not-so-long-range.md %}). One question was why we don't use AI to analyze our extensive logs, given that they currently require a lot of domain knowledge to fully understand.

The main problem is: they are freaking huge. They are designed to be human readable and contain all the information that may be needed to debug issues. From low-level serial traffic, to comments on high level application flow. For busy networks, gathering hundreds of megabytes per day is not uncommon. [Paulus Schoutsen](https://github.com/balloob) in his infinite wisdom joked:

> _"Just GZIP it!"_

Well... he was not that wrong. But more on that later.

## It all depends on the context

So, why is the log size a problem? Most state-of-the-art AI models don't have that large of a context window. And those that do are prohibitively expensive to run, especially if you also expect them to run quickly and give useful results.

![Context](/images/is_4_a_lot.jpg)

AI models operate on so-called tokens. What exactly they represent depends on the model, but you can estimate a token to be [roughly 4 characters of English text](https://help.openai.com/en/articles/4936856-what-are-tokens-and-how-to-count-them). Let's say we want to analyze a 100 MB log file - that would be about 25 million tokens.

Top (AI) models have somewhere between 128k and 1 million tokens of context and cost somewhere between $0.20 and $5 per million input tokens to run. So if we wanted to analyze the whole log file, that would be worst case 200 API calls and up to $125 in costs. And forget about running this at home, unless you have a supercomputer at your disposal.

## Copilot to the rescue?!

I was curious though what it would take to make this work. Luckily, as a contributor to popular open source repositories, I have free access to GitHub Copilot Pro. This means I can give Claude Sonnet 4 a couple hundred tasks per month for free. I also experimented with Burke Holland's [GPT 4.1 Beast Mode](https://gist.github.com/burkeholland/88af0249c4b6aff3820bf37898c8bacf), but it is just not that good at following instructions.

These models can be run in agent mode, which means they can take a task and then break it down into smaller subtasks that they can execute sequentially. This is perfect for analyzing large logs, as it allows the model to process them in chunks that fit into the context window. At least in theory.

Despite my best efforts with the prompt...

```
[...]

Log files are very long and do not fit into your memory.
You MUST read them in chunks of 2000 lines at a time.

Analyze each chunk, immediately answer the question for that
chunk, and then continue with the next chunk. Process the
entire file and do not stop before you reach the end.

It is of utmost importance that you follow these rules, as
they ensure that you do not miss any important information
in the log file.

Before ending your analysis, make sure you have read the
entire file and processed all chunks.

If applicable, you may summarize the results of your analysis
at the end, but do not do so before you have processed the
entire file.
```

... they would sometimes be lazy, and just read the first chunk, or read nothing at all and make up answers. Especially GPT 4.1.

If they did what they were told, the response was actually pretty good most of the time. So I had _some_ hope.

## MCP! 🏁 MCP! 🏁 MCP? 🏁

I then tried a different approach. Instead of instructing the model with language and letting it read the file, I gave it access to a tool that would do the reading for it. This was done by implementing a small CLI utility that I registered as an [MCP server](https://en.wikipedia.org/wiki/Model_Context_Protocol) in VSCode. The agent would simply instruct the tool to give it the next chunk of the log file and whether the end of file was reached.

This worked significantly better. Especially Claude would read the file to the end, compress its context a few times in between and give a useful answer at the end. However the speed was around 100 lines every 10-15 seconds - a no-go for large files. I have personally analyzed files with over a million lines before...

## Did someone say compression?

Soo... GZIP after all? Well, not quite. But our logs do GZIP very well. This is because there is a lot of repetition in them - timestamps, white space, regularly executed commands, transport encapsulation that is part of a lot of communication. And what compression algorithms essentially do is to replace repeated patterns with shorter representations. And since our main problem is that our logs in their raw form are too large for AI models to handle, we actually need some form of compression.

You see, good logs were one of the first things I worked on when I wrote Z-Wave JS. I had a very small number of devices at the time, and I knew I would be reading users' logs a lot. So I needed to make sure that they were easy to read (if you have the technical background to understand them). This includes rendering nested structures, line breaks in messages, etc. For example, these two entries already tell me a lot:
```
2025-08-01 14:33:57.500 CNTRLR   [Node 257] [~] [Multilevel Sensor] Air temperature: 21.5 => 21.6  [Endpoint 0]
2025-08-01 14:33:57.501 DRIVER « [Node 257] [REQ] [BridgeApplicationCommand]
                                 │ RSSI: -107 dBm
                                 └─[Security2CCMessageEncapsulation]
                                   │ sequence number: 208
                                   │ extensions:
                                   │ · type: SPAN  sender EI: (binary blob)
                                   │ security class:  S2_Authenticated
                                   └─[MultilevelSensorCCReport]
                                       sensor type: Air temperature
                                       scale:       Celsius
                                       value:       21.6
```
Node 257 reports a new air temperature reading of 21.6 °C and it previously reported 21.5 °C. I see that it has a weak signal (-107 dBm is close to the sensitivity of 800 series chips), that it was sent encrypted with Security S2 on the `S2_Authenticated` security class, and that its encryption state needed to be re-synchronized. I can also check the sequence number against previous reports to see if the device is sending duplicate reports, or if a report may have been lost.

But there is also a lot of white space in just this short snippet, more than half of the characters. In addition, logs contain a lot of information that's completely irrelevant for analyzing it with AI, like raw serial traffic. Lots of room for compression. So what we do now, is to parse the formatted logs back into a structured format, assign some semantic meaning to the entries, filter out the irrelevant parts, and then serialize them as JSON, one entry per line. This turns the above logs into:

```json
{"kind":"VALUE_UPDATED","timestamp":"2025-08-01 14:33:57.500","nodeId":257,"commandClass":"Multilevel Sensor","endpointIndex":0,"property":"Air temperature","prevValue":21.5,"value":21.6}
{"kind":"INCOMING_COMMAND","timestamp":"2025-08-01 14:33:57.501","nodeId":257,"rssi":"-107 dBm","payload":{"message":"Security2CCMessageEncapsulation","attributes":{"sequence number":208,"extensions":"","· type":"SPAN  sender EI: (binary blob)","security class":"S2_Authenticated"},"nested":{"message":"MultilevelSensorCCReport","attributes":{"sensor type":"Air temperature","scale":"Celsius","value":21.6}}}}
```

With all the filtering in place, a typical log file gets compressed to about 30-35% of its original size. This is still a lot, but it is enough to comfortably fit the log file for a bit more than 24 hours of a smaller network into a 1M token context window. And with the additional semantic information, the AI analysis is much more accurate and useful.

## Taking it for a spin

To demonstrate, I took a log file where one buggy device was sometimes reporting nonsensical values (0 °C temperature in the middle of summer), and asked the AI:

> Why is my (non Z-Wave) heating system randomly turning on in the middle of summer?

The response? See for yourself:


> Based on the log file analysis, your heating system is likely being turned on by an automation that is triggered by a Z-Wave temperature sensor, specifically Node 6, which is periodically reporting an erroneous temperature of 0°C. A temperature of 0°C in the middle of summer would logically cause a heating system to activate if it's automated based on that sensor's readings.
> 
> Here are the instances in the log where Node 6 reported a temperature of 0°C:
> 
> [...]
> 
> **Cause of the issue**
> 
> Each time Node 6 sends these incorrect 0°C readings, it's immediately after a communication problem where the secure connection between the controller and the device had to be re-established. The log shows a `Security2CC_CannotDecode` error, followed by a security negotiation, and then the erroneous temperature report. This behavior suggests a potential firmware issue in the device (Node 6), where it fails to properly resume its sensor readings after a communication disruption and sends a default/uninitialized value of 0.
>
> [...]

**Pretty damn accurate** - and I didn't even explain the encryption part in the system prompt yet!

## Try it yourself!

Once I got this to work, I made the log analyzer into a web-based tool that you can use yourself:

[Z-Wave JS Log Analyzer](https://zwave-js.github.io/log-analyzer/)

It will feel somewhat familiar to users of other AI chat bots, and you can ask followup questions to your log. For now, we use Google Gemini 2.5 Pro under the hood, because it has the required context size, and equally important: a free tier. All you need is to create a free API key and configure the logfile analyzer to use it.