---
layout: post
title: "Working Notes - 12/02/26"
---
Busy week so I’m just going to talk about Anthropic’s new release – Opus 4.6. I will note with a raised eyebrow that GPT5.3 was released within 45 minutes of Opus – I guess OpenAI are feeling pretty threatened at the moment!

## New model, new capabilities
The main changes for [Opus 4.6 ](https://www-cdn.anthropic.com/c788cbc0a3da9135112f97cdf6dcd06f2c16cee2.pdf) are the normal 'more intelligence' but also a larger 1M token context window and the ability to tune the level of reasoning from low to max. The pricing remains the same as for Opus 4.5, although you can now pay (a lot) more for [‘fast’ mode](https://code.claude.com/docs/en/fast-mode).

For the past few releases it's been clear that in order to actually experience the power of the newer models you need to be using them in an agentic framework such as [Claude Code](https://code.claude.com/docs/en/quickstart). This is for the simple reason that it's hard to stretch the capabilities of the models with a straight forward Q&A session in a web chat app.

I’ll briefly sketch out a few examples of how I’ve been using it this week:
- Worked through a problem I had about how best to create a (sort of) data dictionary whilst only having access to WebI-generated SQL queries - it suggested a conceptual approach I'd not considered which does indeed appear to work. Note that although I did this within Claude Code the task was entirely conceptual with no code or any data at all - just a description of the problem. 
- Created a revised draft of an external presentation I'd delivered a few months ago, again - no code involved, but I got back an extremely decent attempt along with some great speaking notes (I just gave it a bunch of these blogs and told it to make it sound like me). 
- Used it to do a few python related coding tasks - these it one-shotted. None of them were particularly complicated, and Opus 4.5 would probably one-shotted them as well, but it’s amazing how far this has come even from Sonnet 4.5.

These are just some isolated anecdotes, but having had a week to play with it, my TLDR is that a lot of the hype is real. When used in Claude Code it does seem better than previous versions at self-critiquing its answers and improving them without further prompting – also I notice my expectation of coding problems is now that I’ll probably get a good answer on the first go. All in all it does seem to be cleverer than Opus 4.5. And let us not forget that 4.5 was already very clever indeed. 

## What could go wrong?	
Anthropic have released the model as an ASL-3 model (AI Safety Level 3). This is a level defined within their [safety framework](https://www.anthropic.com/news/anthropics-responsible-scaling-policy) as a model "that substantially increase the risk of catastrophic misuse compared to non-AI baselines". 

In this same framework document they state they will provide a definition of ASL-4 before a model that reaches ASL-3 is released. Whilst they have done this at a high level for AI R&D and biological risk, the thresholds remain qualitative even if the evaluations behind it are detailed. For biological risk specifically, they've simply put it as "the ability for a model to substantially uplift moderately-resourced state programs" which I guess is fine as publishing a more detailed criteria could in itself be dangerous (e.g. if you published a list of the very specific things that makes biological weapons tricky to make, you're kinda providing an instruction book). The AI R&D assessment is based on a survey of 16 of their technical staff. Again, this doesn’t seem too bad – I’d expect those employees to know if they’re in imminent danger of being replaced. 

However, for cyber threats they have explicitly not provided an ASL definition at any level whilst simultaneously being certain that they are not at ASL-4 yet. This seems a bit weak - they are essentially saturating all of their automated ASL-3 benchmarks at this point. Whilst this is markedly better than say Deepseek, who are on record as saying they don't have [compute to spare for safety work](https://www.scmp.com/tech/article/3342279/chinese-ai-firms-defend-safety-practices-push-back-western-criticism), it does appear to highlight a gap in their own published standards. To their credit, Anthropic are not blind to this criticism and have released the model with ‘additional safeguards’ in a number of cyber related areas (e.g. agentic coding use).

This really matters, as I think it's fair to say that if Opus 4.6 is manipulated to circumvent its guardrails in the same way that [Claude Code was used in September](https://assets.anthropic.com/m/ec212e6566a0d47/original/Disrupting-the-first-reported-AI-orchestrated-cyber-espionage-campaign.pdf) then it's likely a potent cyber threat to enterprises large and small. Indeed, as part of the release hype, Anthropic have posted a blog about finding [500+ zero day exploits](https://red.anthropic.com/2026/zero-days/) - I am somewhat sceptical about how much weight to place on this as no CVE details have been released, but there is little doubt AI assisted hacking is already a thing.

There is positive news on prompt injection - this is basically now 0% success in coding tests, though given it's already been jail broken to extract the [system prompt](https://github.com/elder-plinius/CL4R1T4S/blob/main/ANTHROPIC/Claude_Opus_4.6.txt) perhaps they need some harder tests. Slightly less rosy on the indirect prompt injection, but at least it's much improved on the key enterprise use case of browser use (aka can I get this thing to update my legacy system that doesn’t have an API) - this has a success of only 0.08% of attempts in their testing with their safeguards in place.

Reading the above back, this does come across as a little negative. However, I want to stress that this is an amazing model, and I'd encourage everyone to go and use it inside Claude Code. Most of my cyber safety concerns are just as valid applied to OpenAI and other model providers (if not more so!). The next year is going to be quite challenging!

## Other stuff
New model releases aside – here in no particular order are some other items that caught my eye this week:
- AI is cool, mars is cool, robots are cool, so [the Perseverance rover being driven around the surface of mars by Claude](https://x.com/AnthropicAI/status/2017313346375004487) is triple cool.
- Has Google cracked the holy grail of enterprise AI use cases? [Automated calendar scheduling that takes into account everyone’s availability](https://x.com/omooretweets/status/2018819920893792383) – I will pay good money for this in Outlook.

