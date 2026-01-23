---
layout: post
title: "Working Notes - 23/01/26"
---

A busy week at work but managed to find a little bit of time to get Claude Code to implement a PoC of the Recursive Language Model approach - shared some findings below. Also some notes on tech trends for 2026, and a review of LLMs as judge approaches.

## Adventures in RLMs

[Last week](https://ewanpanter.github.io/2026-01-15/Things-I've-noted-this-week) I wrote about recursive language models. I've spent a bit of time playing around with Claude Code, and have got it to build me a proof of concept of the concepts in [the paper](https://arxiv.org/pdf/2512.24601). This took a little more than 'build this paper' but if I’m honest, not **that** much more. Claude Code with Opus really is very impressive.

I ended up building two Gemini-based versions of the tool which I've been testing on various horrific [government framework agreements](https://www.crowncommercial.gov.uk/agreements/RM6116). These have the advantage of being public domain, very complex, and somewhat similar to the types of documents we see at work. The first version was a purely deterministic version that took the concept of dumping everything into a variable and chunking it with subagents, and the second implemented the full REPL approach with code execution and code.

The results were interesting - both versions are excellent at queries that need all of a document to be examined and would fail with a traditional RAG approach (e.g. find all the definitions across multiple documents). The performance of the REPL version very much depends on the model used for the orchestrator - if you use 2.5 Flash you get a very similar result to the more deterministic approach. However if you use a Pro model, you tend to get a lower number of sub-agents (50 in a typical task vs 70) as the model uses the tools (Regex etc) to delegate more effectively.

The TLDR is that this is an excellent approach for long-context analysis of complex queries that need to be broken down, but I found a heavily scaffolded version was often as effective (and often more token efficient due to fewer reasoning steps) as the more free form REPL approach. When it was more effective, this was very dependent on using a larger more expensive model as the orchestrator. I'll keep playing around with it and perhaps give the sub-agents their own REPL environments - something the original paper did not attempt to explore.

## Tech Trends 2026

A very [readable report](https://www.cbinsights.com/reports/CB-Insights_Tech-Trends-2026.pdf) from CB Insights on possible tech trends for 2026. They start with back office agentic automation where it sees FinTech as leading the way due to the document heavy environment. Whilst this does seem like a real trend, I’m pretty sceptical of the self-reported percentage of organisations that have deployed agentic AI (36% apparently) given that a lot of things can be tagged as 'agentic' to senior leadership (not least MS CoPilot!). 

I think robotics is an interesting one - there have been impressive advances in [world models](https://deepmind.google/blog/genie-3-a-new-frontier-for-world-models/) this year, and LLMs seem a obvious thing to plug into robotics to give higher level planning. However, given the degree that effective agents have to be scaffolded at the moment it'll be interesting to see if anyone can crack robotics in non-highly controlled environments which don't lend themselves to strict rules. Assuming progress is made, it'll doubtless bring with it the need for low latency networks and edge AI.

Sovereign AI is also called out as a key trend. Given the geopolitics of the moment I think this is a no-brainer. Countries are going to want to be assured of their AI capabilities (both hardware and software) as these become more critical to businesses and government. Europe obviously has the additional incentive of compliance with the EU AI Act and GDPR rules. That being said I imagine decision makers will remain pragmatic as long as the frontier capabilities (and data centre capacity) exists only in the US.  <!--more--> 

## Don't judge me

We're all familiar with the LLM as judge paradigm, [this paper](https://arxiv.org/pdf/2601.05111) is a survey of the more advanced agent as judge approach. In an LLM as judge you use the LLM to assess whether a response (or test / eval / whatever) is correct in a single pass. The agent as a judge is exactly what it sounds like - you provide an LLM with tools and it executes a loop using the tools (e.g. search, or a python environment to test generated code) to assess the validity of the response.

The paper draws out three basic approaches. Firstly a rigid procedural based approach (what I’d normally call scaffolding) where the agent follows a set script to assess the response. Then they have the more flexible 'reactive' approach where the agent has a pre-defined set of decisions it can make - e.g. trigger search if it's a factual response, or to run a python environment if it's code. Finally they have what they call 'self-evolving' approach which aims to have the agent have a freer hand about how it evaluates the response (e.g. creating new measurement criteria) and has a memory file that can store previously provided feedback or heuristics. 

The first approach resonates with me - this can be made pretty robust, the second approach seems a bit pointless - on most occasions you know the type of expected response, and the third one I'd be sceptical of being at all reliable at scale without lots of scaffolding which somewhat defeats the point. As with all agent based approaches, this is particularly token intensive - to have any chance of getting the self-evolving approach to work you'll be using Opus or Gemini 3 Pro which will get pricey fast at scale. Give it six months tho...

## Cowork Hype

Judging by the hype Claude Cowork is going to be a big deal, it's now [available](https://claude.com/blog/cowork-research-preview) for Pro users as well as Max users - alas only for macOS. The excellent [Odd Lots podcast](https://overcast.fm/+AA5AWOwPyEo) has covered it, along with the [WSJ](https://www.wsj.com/tech/ai/anthropic-claude-code-ai-7a46460e), and endless threads on [Reddit](https://www.reddit.com/r/ClaudeCode/comments/1qh78yf/tried_claude_cowork_last_night_and_it_was_a_top_3/). The consensus generally aligns with my thoughts from last week - the capabilities were there in Claude Code already, but the less technical interface has exposed it to a wider group of users. This is reminiscent of the [R1 moment](https://www.reddit.com/r/DeepSeek/comments/1qgy3lk/one_year_since_the_deepseek_moment_the_impact_is/) almost exactly a year ago - the technology was there for a while but the general population wasn't exposed to it and when they are there is a collective 'wow' moment.

## A couple of other things that caught my eye

A [handy tool](https://github.com/jaswsunny/youtube-to-text) to convert any youtube podcast/video into a clean nicely formatted transcript - worth it for the WindowsXP interface alone. Putting in a music video is quite amusing for the summary. 

X (do we have to call it that now?) have open sourced [their algorithm](https://github.com/xai-org/x-algorithm) resulting in an analysis, the TLDR is that it rewards engagement and if you put links in that go off X it'll mark you down. The weight values remain a mystery so it's less interesting that it sounds.