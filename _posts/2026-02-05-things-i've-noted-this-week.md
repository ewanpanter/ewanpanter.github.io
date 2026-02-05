---
layout: post
title: "Working Notes - 05/02/26"
---
A mix this week - thoughts on research that shows getting AI to write unfamiliar code means you learn less, bringing interactivity to MCP, and a clever RAG technique that's worth adding to your experiment pile.

## GPT 4o still doesn't make you faster

There is a meme on X that essentially says that every time there is a paper that states that AI doesn't make you more productive, you invariably look inside and find out they used ChatGPT4 in a sidebar. Whilst that is probably doing [this paper](https://arxiv.org/pdf/2601.20245) something of a disservice, when you look inside, you find that yes - they are using 4o in a sidebar. 

Leaving that aside it's still quite an interesting paper as the framing is less about productivity (although that is commented on) it's about learning in junior software engineers. Specifically they find that when two groups of engineers are asked to learn and take a quiz on an [obscure software library](https://pypi.org/project/trio/), the group with access to the AI scored lower (approx. 17%) on the quiz and were also not really any faster (p=0.39 for the stats fans).  

In their interpretation they split the AI group into two buckets. Firstly the high scorers, these tended to either use the AI to ask conceptual questions, or used the AI to generate answers which gave code and explanations. The lower scorers were on the other hand inclined to just YOLO it and fully delegate to the AI, either immediately or after the first task, or those who iteratively debugged the code by sticking the error straight into the AI and saying 'fix it'.

In many ways this is unsurprising - they found that if someone engages with the problem and understands the solution then they learn something. If on the other hand, if you get a tool to do it for you, then you don't learn much. An analogy might be that if you use a calculator to do long division you're not going to get good at long division. 

This analogy then prompts the question - does it matter? 

The authors argue that it does matter as you need someone to verify the answer. I find that less than compelling. It matters not a jot in my life that I'm appalling at long division - in any likely situation where I’ll need to do it, I can pull out my phone. Likewise, if the study subjects had been equipped with a modern agentic system like Claude Code I expect they'd have learnt even less, but they'd undoubtedly have been a lot faster, almost certainly got 100% in the coding task and the agent would have written its own verification tests. Is there any likely scenario where a junior engineer is going to progress through their career without the benefit of an agentic AI assistant? I find it more likely than not that most (though not all) engineers will not be writing much code, if any, in 5 years’ time - I expect they'll be directing swarms of coding agents if not replaced entirely. This is not without precedent – people can still make jam, or knit their own clothes, but these are now hobbies rather than professional necessities.

Beyond the narrow coding example, these findings definitely have wider implications, particularly for educators who need to work out how we can use the tools to enhance rather than supplant learning. That's a bigger question than I can answer in my blog, but it certainly makes me ponder how education is going to pan out for my two young kids. 
On a tangential but related point, we may also see a lessening of the open source ecosystem - the authors used an obscure library in the example, but in the future world are we going to have similar obscure libraries to use in such studies? The agents are going to go to what they know, and if it doesn't do what they want, they'll code something new. After all, open source libraries exist to prevent people reinventing the wheel, agentic systems are quite happy to reinvent the wheel and won’t have the same incentive to share their code and learnings with the world in the same way as today’s engineers. The better the agents get, the bigger these problems.

## MCP Apps

[MCP](https://modelcontextprotocol.io/docs/getting-started/intro) has been an open standard for over a year now and it's now got an official extension - [MCP Apps](https://blog.modelcontextprotocol.io/posts/2026-01-26-mcp-apps/). In a nutshell this allows a custom interactive UI to be displayed within a chat window via an embedded iframe.

For use cases where you're going back to the human with some results from the MCP query, this is pretty useful. For example, let's say your MCP server can query your BI system and you've pulled back some sales stats, this can now be displayed as a chart inline with the chat window. The inline window can then interact directly with the MCP server without going via the LLM - if you want to filter by geography you just use the dropdown. This both saves on tokens, but importantly makes it a deterministic process so you're not worried that some weird UI will be generated or the LLM will hallucinate filtering details. Obviously, this can make the model out of sync with what you've just done to the view (e.g. you've filtered on Europe) but there is provision in the extension to allow you send back a (short) message to give the model updated context (e.g. the user filtered on Europe).

Like the rest of MCP this is a build once use many situation - the UI will work in Claude or ChatGPT or any software that supports the extension. This already includes VS Code, so potentially you're going to get things like interactive diffs in the chat which has the potential to be quite useful. Importantly it's also a progressive enhancement, if the client doesn't support it, the MCP tool will still work.

Whether this will take off remains to be seen. The main blocker is the walled garden incentive - MS wants to keep you in their Copilot world, so are going to prefer you to build a Copilot connector, but I think we'll probably see some adoption where it makes sense (e.g. I expect Salesforce will add it to their MCP server). When combined with the [Agent Skills](https://agentskills.io/home) you've got a pretty powerful combination - instead of having text prompts at points of human interaction, you can now have interactive elements.<!--more--> 

## Improved chunking
One of the challenges with naïve RAG is deciding the size of the chunk - small means just specific facts are found, large means you get more context but you'll also include unneeded information (plus the semantic signal is going to be less specific - since embedding is a kind of compression). Quite often people just YOLO it and go for some arbitrary size like 200 tokens. 

The authors of this [blog post](https://www.ai21.com/blog/query-dependent-chunking/) offer a new approach and suggest that there is no universally suitable chunk size for a given task, and instead text should be chunked at multiple different sizes. 

Their approach then uses [Reciprocal Rank Fusion (RRF)](https://dev.to/master-rj/understanding-reciprocal-rank-fusion-rrf-in-retrieval-augmented-systems-52kc) with the constant set up to mean that a single chunk size will tend to dominate the rankings. 

In summary, each chunk is recorded with meta-data (document id (e.g. document A) and character range (e.g 4000-4150). The semantic search is done on all the chunk size indexes and the individual results ranked in the normal best to worst way. The key change is that the parent documents for each chunk are then ranked so that documents that appear multiple times and/or rank very highly in their sub-lists will end up with an overall document ranking. The authors are only covering the document retrieval aspect, but in practice you'd then use the top ranked chunks as an anchor to determine which part of the winning document to put in the final context window.

The actual material impact of this approach does seem to vary – for most of the retrieval benchmarks they tried it was between 1-5% improvement over naïve RAG, but for one result (TRECCOVID) they got a 37% improvement. I suspect this means that some datasets / query types particularly lend themselves to this approach but for a mature pipeline even 1-5% improvement is not bad.  

However, much as I appreciate the technical neatness of this technique, its variable results probably mean it’s one to put in the ‘experiment’ pile rather than the ‘try and rush into prod’ pile.  A key point of this technique is that it doesn’t require anything particularly special (no retraining etc) just a different approach to selecting the top k results. The cost is of course multiple vector indexes but assuming you are executing the searches in parallel, that is only a storage cost and an embedding cost - both of those are cheaper than giving the wrong answer to your user. 

If the chunking size you’ve picked for a mature RAG pipeline is working well, then great, if not then experimenting with multiple chunking approaches such as this one is probably worthwhile – especially if corporate security rules make introducing a model-based re-ranking model such as Cohere tricky.
