---
layout: post
title: "Working Notes - 19/02/26"
---

This week I wanted to cover one of the interesting things that came out of Anthropic that wasn't [Sonnet 4.6](https://www-cdn.anthropic.com/78073f739564e986ff3e28522761a7a0b4484f84.pdf).

## The future is not evenly distributed
Anthropic release a lot of interesting information into the public domain, this week they've published a blog about [agent autonomy](https://www.anthropic.com/research/measuring-agent-autonomy). The TLDR of the blog post is that they're mainly looking at how long 'turns' (how long an agent runs for before asking for human input) last for in Claude Code (Anthropic’s agentic coding tool) and on the Claude API, and they find that the length of the longest turns have been increasing from 25 minutes in October 2025 to about 45 minutes now. 

The immediate thought is well, yes you would see that since the models have increased in quality. However what you don't see is immediate jumps after a model release, the graph is somewhat bumpy but the trend is fairly smooth. For Claude Code the authors point to a number of reasons for this, principally the Claude Code agentic harness [has improved](https://code.claude.com/docs/en/changelog), but also users are trusting it more as they become more experienced. This is an interesting observation and I buy it based on my own experience. It took me quite a few goes with Claude Code before I realised what the tool was actually capable of – I’m pretty certain I’m still not using it to full capacity!

For API calls the blog also shows what users have been using agent functionality on the models for and plot this on a risk / autonomy scale. A high risk/ autonomy task might be someone using the API to make financial trading decisions, whereas a low risk / autonomy task would be like someone using it to complete simple calculations. What would be really interesting, is whether this is changing over time – unfortunately this part of the analysis is based on a snapshot of data, although they do say they'll repeat in the future so perhaps this can be derived. I would expect that as people’s comfort level with the tool goes up with use, they will also start using the tool for high risk/ autonomy tasks.

I also find the article interesting for something it doesn't cover - the diffusion of knowledge about this technology through enterprises. It's increasingly obvious to me there is a substantial epistemic gap between people who have used tools like Claude Code and those who have only used the free tier of say ChatGPT 4o or MS Copilot (which in many enterprises will still be using 4o under the hood). To quote William Gibson, 'The future is here it's just not evenly distributed', is very much true when it comes to AI tools at the moment.

Those with unrestricted access (i.e. the latest models) to tools like Claude Code / Cowork are having a very different experience to those who are using Copilot in Outlook to give tone coaching. There are good reasons why this is so - for example the bar for enterprise security and privacy is rightly higher due to consequence and complexity. But this does not change the fact that the gap is very much there and that it's likely impacting strategic decision making in enterprises.

Furthermore, it's natural to use an existing mental model to frame this technology. To a first approximation one CRM system is very much like another - there are nuances but they do the same thing. This is not true for current AI. There seems a substantial risk that applying a commodity mind-set will result in a company being less competitive. The article is discussing systems that will run autonomously for 45 minutes plus, create their own tests, create pull requests, and self-correct. This is a different class of product with potentially profound operating model impacts. At the very least serious consideration should be given to how this type of capability can be embedded in an organisation and what priority we should put on getting agentic systems through our security / privacy processes.

Until more people (and especially leaders) within enterprises have experienced the difference between state-of-the-art agentic harnesses and simple chatbots / autocomplete the strategic conversation is going to be stuck at the wrong level. 
 
## One more thing

One of the things that makes agentic systems so powerful is their ability to apply [agent skills](https://agentskills.io/home) as they allow agents to behave in a reliable / structured manner. Anthropic's new [guide to building skills](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf) is an excellent little resource and definitely worth a read – and trying out!


