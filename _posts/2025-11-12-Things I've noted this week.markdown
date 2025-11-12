---
layout: post
title: "Working Notes - 12/11/25"
---

## More progressive exploration
[Anthropic is offering better practices for tool use within MCP](https://www.anthropic.com/engineering/code-execution-with-mcp). Instead of loading the tool into context window, the model can progressively explore as required and write code to access the tool which is then executed in a sandboxed env. For example if we had a tool that wanted to use a large text file, the model would write code to operate the tool on the large text tile in it's file system rather than loading the large file into the context window. This progressive discovery of tools and execution on a virtual file system does seem to be 'the way' and is very simuilar to anthropics other recent work - Claude skills (which I love btw).

## A telco benchmark you say? With a dual sided agentic test harness? Tell me more!
Sierra Research has made a telco agent benchmark called [tau2-bench](https://github.com/sierra-research/tau2-bench). What they've done is created a test harness which provides a simulated world both the agent being evaluated exists in but also a simulated user - this is quite novel, either party can make a change via tools and the other side would see that change - e.g. if the agent suggests you're in airplane mode which is why your data isn't working, the user can toggle airplane mode on on a simulated mobile phone. You can either just use their reference agent architecture with whatever LLM you want to evaluate, or you can swap the agent out for your own customer service agent (assuming you hack it to output the required format, which tbf is pretty generic). The user is always simulated by the same LLM (for some reason they've chosen GPT4.1 which is somewhat bold!). Clever stuff. You can look at the leaderboard [here](https://artificialanalysis.ai/evaluations/tau2-bench) or read the paper here.

## EU Kicking the can down the road
Widely [reported](https://www.ft.com/content/af6c6dbe-ce63-47cc-8923-8bce4007f6e1) that the EU is going to water down the EU AI Act. I have mixed feelings on the act - being usecase focused I think there is a lot of scope for constraining innovation. From my experience in the real world, it's certainly a chilling effect on AI implementation. The current reporting suggests that most of the rules for items defined as high risk within the act will be delayed by a year and they won't fine anyone for a further year after that. In a lot of ways that just further muddys the water by kicking the can down the road - AI is likely to be quite different in two years time - the rules might not even make sense then.
<!--more-->
## Other stuff
[Kimi K2 Thinking](https://moonshotai.github.io/Kimi-K2/thinking.html) is out from Moonshot AI as predicted by the entire world when they released Kimi k2. Seems to be a good model, though a few people are suggesting taking the benchmarks with a pinch of salt as they may have been benchhacked (may have just made that word up!). From my very quick play, seems decent but worse than GPT-5 and Claude Sonnet 4.5 when I tell it to do my standard 'is my council tax too high' research project. [Have a play here](https://www.kimi.com/chat/).

Googles stuck gemini in [google maps](https://blog.google/products/maps/gemini-navigation-features-landmark-lens/) but only in the US so who knows if it's any good.

