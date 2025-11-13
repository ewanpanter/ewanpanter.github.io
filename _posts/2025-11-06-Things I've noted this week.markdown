---
layout: post
title: "Working Notes - 6/11/25"
---

Lot's of things to cover this week - new models with terrible names and a deep dive into the water use of AI.

## Does AI use incredibly amounts of water?

Andy Masley offers an in-depth look into [how much water AI uses](https://andymasley.substack.com/p/the-ai-water-issue-is-fake). Spoiler, quite a bit but then so does everything else. I find it quite useful when thinking about big numbers in the news to channel my inner [David Spiegelhalter](https://x.com/d_spiegel) and ask 'is that a big number'. For example, the [water footprint](https://watercalculator.org/) of a pair of jeans is nearly 11,000L which is the equivilant of approximately 5.4m prompts (you can argue that number up or down an order of magnitude, but that doesn't alter the point). As ever context is everything (pun intended).

## Awful name good model
MiniMax (yet another Chinese AI company) have released a new open weights model called [M2](https://agent.minimax.io/) (MoE with 10B parameters etc etc). Gets good benchmarks for a open weights model. I've not properly used it in anger, but gave it a complex task (compare my houses council tax valuation with nearby prosperities) and it gave it a decent stab. Minimax is a Chinese based company so corporate privacy and security (and implementing the TSA) may rule it out for actual prod use even if self hosted.

## Anthropic is corporate number one
Anthropic has seemingly overtaken OpenAI in corporate LLM token use according to [various](https://seekingalpha.com/news/4505254-openai-dominating-consumer-ai-token-consumption-anthropic-wins-enterprise-barclays) [sources](https://x.com/StefanFSchubert/status/1982688358561439759). This is pretty interesting, and doubtless driven by use of Claude Code. I'm a big fan of the Claude models, esp the new Claude skills functionality. I do note that the articles don't mention whether MS Copilot token use is counted in the API share - I would guess not (not least as MS host their own instances of OpenAI models) and would assume this might change what the graphs look like. Anthropic have also been doing interesting things in tailoring their products to specific industries / corporate usecases - currently on the wait list for [Claude for Excel](https://www.claude.com/claude-for-excel) will be interesting to see if they've come up with a robust way of handling tabular spreadsheet data (as MS Copilot for Excel is not great).
<!--more-->

## Cursor 2 is out
[Cursor 2.0 is out](https://x.com/cursor_ai/status/1983567619946147967). The main news being that Cursor have trained their own model called Composer (tho who knows if it's actually a fork of qwen or something). They've also got a new more agent centric UI option compared to the IDE - does hide the code a bit more than I think they should for a development tool. I should test this more than I have, but I've been using claude code for web recently so.... It's certainly fast tho.

## How to make your AI safeguards actually work
OpenAI's [gpt-oss-safeguard models](https://openai.com/index/gpt-oss-safeguard-technical-report/) are pretty interesting. One of the issues that corporate users face is safety classfication, esp if the model input or output is exposed to customers. From my own experience roll your own classifiers can be problematic (cost / false positives) so having a small quick option is certainly interesting. The devil is in the detail tho, if you read their docs you can see that they're suggesting that the polices should be about [300-600 tokens](https://cookbook.openai.com/articles/gpt-oss-safeguard-guide) - to me that seems a bit small knowing the ability of the corporates to allow everyone their two pence on what is / isn't allowed. That being said, it has the obvious advantages of a) OpenAI are probably going to be better at making a classifier than you are b) it's quick and c) you can look at the CoT to find out why it's made it's determination. Note, it's not multimodal - I don't think I've seen a good multimodal saftey classifier solution yet. Done in conjunction with [ROOST](https://roost.tools/) who are a non-profit that aims to help build open source tools for online safety (worth checking them out).

## Other stuff and papers I read
OpenAI have changed from a non-profit to a public benefit corporation (note that a public benefit corporation does not benefit the public as much as you might think). Lots of reasons for this (all revolving around making money), but to me at least, it does seem that the non-profit has been a bit short changed here - sure they get a lot of money, but given they controlled the entire thing beforehand, it did seem like they were owed more... Maybe it won't matter but we'll see.

Anthropic continue to publish interesting papers on the [inner workings of models](https://transformer-circuits.pub/2025/introspection/index.html). This one is on introspection - they show that the models can understand when they've had a 'thought' injected into during inference (assuming I got the jist, they're working out the vector for some concept (e.g. 'bread') before hand, and then during inference for an unrelated prompt they're adding that vector in). Being able to understand how a model 'thinks' about it's internal state is obviously useful for AI safety and interpretability. 

Finally, this is an [Interesting paper](https://arxiv.org/pdf/2510.23595) on using a model to create tricky questions, formulate an answer, review the answer, and then update the model weights. Essentially self supervised RL - sort of an obvious idea in retrospect, but not one that anyone else had published on. Does seem to work according to the paper - I've used agents to judge the output of another agent, but using the reward signal (i.e. this was a good answer / bad answer) to update it's own weights is obviously much more powerful than just telling it to do it again.
