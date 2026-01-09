---
layout: post
title: "Working Notes - 08/01/26"
---

Back to it after the long Christmas break - I spent most of it with the family and also [remodelling my home office](https://ewanpanter.github.io/2026-01-08/renovating-my-office-with-AI). Things slowed down a bit from a model point of view but this gives us time to talk about data sovereignty and when not to use a vector database.

## Data sovereignty 
I was reading an [interesting article](https://www.theregister.com/2025/12/22/europe_gets_serious_about_cutting/) in El Reg on data sovereignty. The basic premise is that nothing is really sovereign if you're using a US hyperscaler due to the US [Cloud Act](https://en.wikipedia.org/wiki/CLOUD_Act) (tl;dr:  US based hyperscalers can be compelled to disclose data held anywhere). 

This is one of the reasons that the EU-US Data Privacy Framework (DPF) was adopted as it allows US companies to certify they are GDPR adequate. However, it is  straightforward to argue that this [doesn't actually mitigate the problem](https://brianclifton.com/blog/2025/05/19/schrems-iii-how-likely-and-how-to-prepare/). This then leaves technical solutions, for example, if an EU company encrypts their data then it'll be useless even if it is accessed by US law enforcement. Unfortunately, in practice, this is not effective. When requesting keys, GCP attaches a reason code - if this is 'third party access' then access is denied. Sounds good, but if Google was to actually use this reason code then they will breach the gag order that forms part of the request.

A possible solution is to use an external (EU based)  key manager, but this then comes with significant complexity. The long and short of this is that this is something of a nightmare that should be pondered as non-US companies become more and more dependent on US hyperscalers for AI. 

## New SOTA OCR model
Data sovereignty provides a nice segue into [Mistral's new model](https://mistral.ai/news/mistral-ocr-3 ). Being based in the EU, using Mistral can avoid these issues, although you've got to store your data somewhere and if that somewhere is in AWS, Azure, or BigQuery you may not have mitigated the problem!  The new OCR model is SOTA for converting complex formatted documents. In my view, this helps to address a key blocker for AI adoption - huge amounts of important data is locked up in PowerPoints that were formatted to look nice in presentation mode without a thought about whether an AI model would need to consume it down the road. Each improvement to OCR models helps solve that problem and if this issue sounds familiar, it's certainly worth experimenting with in the [Mistral Playground](https://console.mistral.ai/build/document-ai/ocr-playground).

## Context caching and when not to use naive vector based RAG (spoiler - most of the time)
Ever since models like Gemini Flash became good at handling long contexts (the data provided in the prompt) at a low cost, I've held that a lot of implementations of RAG using vector databases are essentially pointless.

To expand on this a bit, the vector DB + LLM paradigm became a thing due to the small context window and high cost of early models ([remember when GPT4 was limited to 8k tokens and $30/m tokens!](https://azure.microsoft.com/en-us/blog/introducing-gpt4-in-azure-openai-service/)). Neither of these constraints exist anymore, and the approach comes with a lot of inherent problems and assumptions - not least that your query needs to be semantically similar to the answer. A lot of the time you can avoid these problems and vector db complexity, just by tossing the entire document set into the context window - the downside being that the larger the provided context the longer the time to the first token.

I've been discussing this with a colleague this week, specifically in the context (pun intended) of using prompt caching instead of RAG with a vector db. As background, a query to an LLM is processed in two phases: the prefill (the heavy lifting where the KV cache is generated based on the context) and the decoding (generating the response tokens). The decoding part is memory bound and the prefill part is compute bound. If you save the KV cache and just reuse it with your new query appended, you can dramatically reduce the time to first token by avoiding a ton of compute. This is prompt caching.

The key constraint to this is that you do need the same prefix prompt - but in a lot of RAG type use cases this is fine, as your prefix are your instructions and some documentation. 

You can obviously make this more effective by having several prompt caches and picking the best one for the query (e.g. using BM25 keyword search or a LLM over a short index saying what each cache contains (you could even cache this if it's long enough!)). Different providers have different approaches (here is [Google's](https://ai.google.dev/gemini-api/docs/caching?lang=python)) but in a lot of cases this is a no brainer - it's cheaper and faster.*
<!--more-->

## Stateful Google API
Currently if you're doing anything clever with agents, or even just a simple chatbot you're generally responsible for managing the state. The state is essentially the interaction history. This can be quite painfully complex when you're dealing with agents and complex interactions.

OpenAI solve this issue via their [Responses API](https://platform.openai.com/docs/api-reference/responses) and now Google has followed suit with their [Interactions](https://ai.google.dev/gemini-api/docs/interactions) API. Instead of storing and supplying the whole interaction history you can now just pass the last interaction id to the API. Additionally, you can also use the API with agents (such as [Deep Research](https://ai.google.dev/gemini-api/docs/deep-research)) and set the agent to work in the background, and come back in a different session to get the results.

In order for this to work Google is storing the state of the model (since you're not anymore) in cloud using their implicit caching approach for 55 days. This is potentially a decent financial saving as the cached tokens are only charged at 10% of their normal cost. Whether that stays that way over the long term... well I guess we'll see - there is an obvious commercial opportunity there! 

This last benefit also has a privacy angle that needs consideration for corporates. We are all used to interactions with the LLM being transient but now Google is storing your data 'somewhere' for 55 days. Yes it's encrypted etc etc, but the privacy team will definitely want a word before going live with this (not least because in the beta you can't specify data residency yet).

## Skills continue to be a thing and are now an open standard
I mentioned in my last entry that it seems increasingly likely that [Anthropic's Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) will be a thing with ChatGPT starting to use it [somewhat unannounced](https://x.com/simonw/status/1999503124592230780?s=20). Well, OpenAI have now [gone official](https://developers.openai.com/codex/skills/) and as with MCP, Anthropic have now made it an [open standard](https://agentskills.io/home). If you've not used skills yet, I'd recommend having a play - they have a lower barrier to entry than MCP and are useful even for casual use. For example, I've made a lesson prep skill for my (primary school teacher) wife that has now saved her a good few hours. I would assume it's just a matter of time before X.ai, Google, and other model providers adopt the new standard.

## Continual learning
If I had to list the things that are preventing truly transformative AI (i.e. models that can replace entire jobs, rather than automate short tasks), top of my list would be continual learning. At the moment we have 'intelligence' but that intelligence can't learn anything after its initial training - yes you can shove ever longer contexts into the model prompt, but this has obvious limits. 

[This paper](https://test-time-training.github.io/e2e.pdf) is a possible baby step towards resolving this. It allows the model to learn from the context as it's generating the tokens. To skip a great deal of technical detail, after each pass, the model weights are updated and only a 8k token KV cache is stored.

This means that the model can then answer questions on the context without querying a massive KV cache making it (in theory) quicker. In theory you can then use the final values of the weights as the starting values of your next query. If you squint a bit you can imagine a developed version of this slowly building up expertise as you ask it more and more questions on existing documents - a sort of continual learning. 

This is clearly not the final answer though. If nothing else, the model will be best at the last document / context it was trained on and forget previous documents from earlier runs. This could be mitigated by freezing the model weights related to the previous query, but then you run into interpretability roadblocks if you want to avoid simplistic assumptions. There is a huge issue of deployability - each customised set of model weights would need to be stored and loaded into memory as and when a user requested that model. 

Tl;dr - interesting but doesn't solve the continual learning problem (although the technique is potentially pretty useful for speeding up long contexts).

## Other stuff
When writing the bit about prompt caching, I came across this [excellent explainer](https://ngrok.com/blog/prompt-caching) from [Sam Rose](https://samwho.dev/) - it actually ends up being a decent explainer of how an LLM works. A highly recommended read. 

As mentioned I've been [remodelling my home office](https://ewanpanter.github.io/2026-01-08/renovating-my-office-with-AI), and before getting my table saw out visualised the result in Nano Banana  - [this](https://minimaxir.com/2025/12/nano-banana-pro/) is an excellent deep dive into Nano Banana prompting.

And finally, in a new fresh hell, AI generated pictures of packages (not) being delivered are now [a thing](https://x.com/ByrneHobart/status/2004734471267103023?s=20). 

* For completeness you need to be aware of cache expiries, as they don't last forever and you'll want to refresh them from time to time.



