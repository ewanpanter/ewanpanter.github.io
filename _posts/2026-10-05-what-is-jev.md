---
layout: post
title: "Jev & Decision Intelligence - What it is, how it works, and what it isn’t"
date: 2026-10-05 20:00:00 +0100
---


# Jev & Decision Intelligence \- What it is, how it works, and what it isn’t

A new AI model named [Jev](https://docs.typesafe.ai/introduction) from a startup named [TypeSafe AI](https://typesafe.ai/) has been getting a bunch of hype recently \- and coined a new term \- decision intelligence. In mid-September when it came out, I glanced at the documentation, stroked my chin a bit, and was a bit surprised that I could guess how it worked. Their lack of an obvious technical moat notwithstanding, it is a super interesting model that fills a real need in the enterprise AI landscape. 

What this post does is firstly explain what decision intelligence is, why it’s useful and then explains how it (probably) works. The second post then documents whether I can make my own on [GCP](https://cloud.google.com/?hl=en) by fine tuning open-weight models (spoiler, I absolutely can) and discusses the performance you can expect if you do. This post starts off high-level and gets geekier as it goes on, so drop out when you reach your nerd threshold\!

## What is decision intelligence?

As in life, there are many enterprise problems where it’s necessary to just make a decision. Is it A or is it B? Which of these options is correct? Score this ticket for urgency. Prior to Jev becoming a thing, the standard approach to this was either to train a [traditional ML classifier](https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.RandomForestClassifier.html) or give the decision to an [LLM](https://chatgpt.com/) (often restricted to a specific format via JSON and [Pydantic](https://pydantic.dev/)).

(Just for the nerds, I’m aware that you could use a modern variant of BERT, but we’re going to ignore that for the purposes of this article \- I don’t think it really adds much)

So, what’s wrong with that you might say \- well, if your data is numerical and tabulated then nothing. You should absolutely go and train an ML model. But if your data is messy (e.g. you want to direct a customer complaint to the correct queue) then using an LLM comes with a number of drawbacks. Specifically:

* Unconstrained output \- the answer is free text. Yes you can address this to an extent by using JSON mode, but then you’re generating a bunch of (essentially) pointless tokens.  
* No usable probability \- when the LLM answers you can get the probabilities of each token it predicted. Naively, you might think that this gives you the probability of the predicted option. But it doesn’t. The provided probabilities are over the model’s entire vocabulary, so if you have options a, b, and c, some of the probability is assigned to ‘the’ or a full stop. It’s not a probability distribution over only the valid options.  
* Cost and latency \- LLMs are [autoregressive](https://en.wikipedia.org/wiki/Autoregressive_model). This means that for each token that is generated it has to do a bunch of matrix multiplication. The model writing “The answer is (a)” or more likely, the same thing padded out with a ton of JSON, will take multiple times the cost and latency of just spitting out “a”. Since cost and time taken scale with output length, you can see the issue.

What Jev (and [its](https://dealroom.co/news/158476-cloudflare-launches-clef-as-typesafes-decision-model-idea-spreads-across/) [copies](https://huggingface.co/blog/sora-2/what-is-openai-decisions-api-a-practical-guide)) does is provide a third approach to answering decision type questions \- it provides a response that is constrained to only the options you give it together with a probability and a value for how confident it is about its answer. It can do this for three types of decisions \- multiple choice (one of N options), score (a position on an ordered scale), and yes/no (exactly what it sounds like, though for [stats geek reasons](https://en.wikipedia.org/wiki/Bernoulli_trial) TypeSafe call this ‘Noul’). 

![Decision AI input and ouput.](/assets/images/261005_jev_image11.png)

Jev is also extremely cheap and quick \- as part of the benchmarking of my homemade version, I found Jev would classify a [publicly available dataset of 1000 customer complaints](https://www.consumerfinance.gov/foia-requests/foia-electronic-reading-room/cfpb-consumer-complaint-database-narratives-archive/), with three decisions for each complaint, for all of 4p in an median time of 126ms/complaint. As a comparison, if you’re really fast at blinking, you’ll top out at 150ms per blink. The same test with [Gemini 3.5 Flash-lite](https://deepmind.google/models/gemini/flash-lite/) cost £1.72 and took about 2.1 seconds/complaint (14 blinks). Accuracy wise there is not much in it \- in my testing, Jev was 3.6 percentage points behind Gemini at 1/40th of the cost in 1/16th of the time.

![Jev is really cheap compared to a normal LLM!](/assets/images/261005_jev_image14.png)

That obviously sounds amazing, but there is a catch. Whether it matters or not very much depends on the problem you’re trying to solve. TypeSafe AI describes Jev as exhibiting ‘System one’ thinking \- this comes from a popular science book named [‘Thinking fast and slow’](https://en.wikipedia.org/wiki/Thinking,_Fast_and_Slow). The basic thesis is that humans do two types of thinking \- quick gut call system one thinking (e.g. is that car getting closer), and slow methodical system two thinking (e.g. [a battle of wits](https://www.youtube.com/watch?v=rMz7JBRbmNo)). This means that whilst Jev is quick and will perform well at certain types of quick decisions it will fail to perform if the decision requires reasoning. 

It’s not as simple as saying ‘Jev is good at classifying customer tickets’ \- it might be, but it depends on how much reasoning is required. If you’ve got two queues and the first deals with complaints and the other deals with password resets, you’ll be fine. However, if you’ve got eight queues, and the only difference between each depends on the diameter of a bearing then maybe you want to treat it cautiously. My own testing underlines this, I found Jev performed terribly where a bit of arithmetic was required to get the right answer.

The important thing to remember is that Jev isn’t really (this is speculation on my part, but I think pretty likely to be true speculation) a new type of AI model \- it’s a small LLM with changes to how inference is performed that has likely been finetuned to make it better at answering decision questions. If your problem can be answered by a small LLM today and a prompt that says “answer with only a single word” then it’ll be fine. If you need [Fable 5.1](https://www.anthropic.com/claude-fable-and-mythos-5-1) to get the right answer \- not so much.

A lot of the hype online has been people saying that Jev solves [hallucinations](https://www.instagram.com/p/DWA2UuQldgk/). It doesn’t. At least it doesn’t in any kind of useful way. Due to the way it works, the answers that it can output are constrained to the options presented. If you give it a choice of A, B, or C, you’re guaranteed that it will pick one of those rather than say ‘Monkey’. What it absolutely will still do is give the wrong answer. So ignore the instagram posts that are saying the opposite. They’re wrong.

Before we move onto how Jev (probably) works, it does have one more trick up its sleeve. With a traditional LLM if you have one piece of contextual text (e.g. a customer ticket \- TypeSafe refers to this as the ‘state’) and you want to ask three questions about it (e.g. Which queue? Priority score? Is the customer angry?), you’d either need to send the context (state) and each question to the model individually, or send the list of the questions with the context and try to split it out using JSON. The second approach often works poorly with smaller models, as they tend to get confused if you ask lots of questions in one go. Jev on the other hand will let you ask dozens or hundreds of (independent) questions in one go without getting any more confused than if you asked a single question.

So the TLDR, is essentially:

Jev is:

- An AI model you can send messages to via an API  
- A good option if your problem can be answered with a quick system one thinking  
- Able to answer multiple choice questions, provide a score on a scale, and yes/no questions  
- Able to simultaneously answer many questions about the same piece of information in a single API call as long as the questions are independent  
- Extremely cheap and extremely fast

Jev is not:

- A good option if your question depends on detailed system two reasoning  
- A fix for the AI being confidently wrong  
- Going to work if the answer to one question is dependent on another

## How does Jev (probably) work?

In order to understand how Jev works, it’s necessary to understand (a bit) about how a normal LLM works. I will keep this fairly non-technical (apologies to the techy folk). Note that all of the below is my speculation \- TypeSafe have not revealed how Jev works. However, since being released loads of copies have come out, all of which basically work in the same way as I guessed \- it seems likely I’m correct. Still, caveat emptor.

When an LLM generates an answer, it first reads in the prompt (its context), and its layers do a bunch of [matrix maths](https://en.wikipedia.org/wiki/Transformer_\(deep_learning\)) (the key ingredient being attention) that results in a list of raw scores across the entire vocabulary of the LLM. The scores are called logits and the vocabulary is every token (word/symbol/number/part of a word/etc) that the model recognises.

These raw logits get tweaked by settings like temperature and usually the unlikely tokens are [chopped](https://en.wikipedia.org/wiki/Top-p_sampling) [off](https://www.geeksforgeeks.org/artificial-intelligence/graph-based-semi-supervised-learning/). This list of scores/logits is processed using a function called [softmax](https://en.wikipedia.org/wiki/Softmax_function) that normalises the distribution to probabilities between 0 and 1\.

From this distribution a token is chosen and spat out by the model to the user. The whole process is then repeated with the new token appended to the original context, and so on and so forth (skipping a whole bunch of stuff like the use of a [KV cache](https://huggingface.co/blog/not-lain/kv-caching) to make it efficient etc). Each round of the process is called a forward pass.

The reason why Jev is so fast and cheap is because it never does an additional round of the process. All the tokens are already known, so the GPU can do all the maths across all the tokens at once, rather than generating multiple tokens one after another.

![Jev never generates any tokens](/assets/images/261005_jev_image8.png)
<!--more-->
To understand how it can do what it does, you first need to understand the way they pass the prompt to the model. TypeSafe separate their LLM prompts into sections as follows:

1. A system prompt that can’t be altered explaining what is going on (this is inferred but I can’t see how else it’d work)  
2. The piece of text that the questions are about \- TypeSafe refer to this as the ‘state’. An example might be a support ticket.  
3. The first question and its criteria \- for example “What queue should this go to? A. Billing B. Complaints”. Each criterion can be provided with some detail and examples.  
4. The second question and its criteria. Perhaps this time it’s a priority score or a yes/no choice.  
5. The nth question. The state can be up to 32k tokens, with as many questions as you like.

![How a Jev request is constructed](/assets/images/261005_jev_image17.png)

Once you’ve understood the way it’s formatting the overall context it’s giving to the model, you can understand the two cunning things that it’s doing.

The first (more simple thing) is how it stops the model choosing an option that doesn’t exist and gives a probability distribution that is guaranteed to sum to 100%. This works by discarding all the logits that are not one of the allowed options, and then running the softmax function over just these valid options (for each question). This means that the model is incapable of returning an invalid option, and that there is no probability leakage to invalid tokens so the probabilities sum to 100% (though as per the first section, this does not mean the provided probability is trustworthy\!). If the question type is a score on a numerical scale (e.g rate from 1 to 5), it treats each level as an option, gets their probabilities via the restricted softmax, and then takes a weighted average to give a decimal score.

![Probabilities and outputs are locked to valid options](/assets/images/261005_jev_image21.png)

The second is a bit more complex, and explains how it can answer \*all\* of the questions at the same time.

Most people assume that an LLM only calculates the logit scores for the next token. It actually doesn’t, it generates an internal representation (which can be easily converted into a logit score) for every token position from the first to the last during the parallel pre-pass. At each position the score would predict the \*next\* token. It’s just that normally, you’re only interested in the \*final\* next token so you can feed it back into the next pass \- so you just ignore all the middle ones as they’re not needed (since you know the actual next token if you’re looking at the middle of a piece of text) .

Jev doesn’t bin them. Instead it pulls the logit scores calculated at the end of question one, then the logit scores at the end of question two, then three, etc. It then runs the softmax on each question's logits on the valid tokens for that question. Et voila \- you have an answer for each question simultaneously. Clever.

![Jev uses the scores at the end of each question](/assets/images/261005_jev_image2.png)

There is one flaw with this plan \- in normal circumstances this would work fine for the first question, but the second question would potentially be biased as it’d be able to see the first question, and the third question would be able to see the first and second questions. There are a couple of ways that this can be dealt with \- the simplest is to use an [attention mask](https://machinelearningmastery.com/a-gentle-introduction-to-attention-masking-in-transformer-models/) \- this means that each question can only see the state (the piece of information that the question refers to) and itself (plus the system prompt). The second approach is the one that Jev probably actually uses (I use both in my clone) which is to read the state once, and then run each question in parallel on top of it. Either way, the parallel calculation of the logits means Jev (and its clones) can get structured decisions back in a hundred milliseconds for a cost of basically nothing.

![Jev probably uses this approach](/assets/images/261005_jev_image6.png)

The final thing to cover is the additional training that Jev has done to make it better at the type of decisions and question formats it is likely to be tasked with. Whilst we don’t know what exactly it’s been trained on, given it’s a startup it’s more than likely that they have applied post-training to an existing open weights model rather than training their own base model. 

The marketing and documentation talks a lot about RLCD (Reinforcement Learning for Calibrated Decisions) but doesn’t actually say what it is. The internet has [speculation](https://www.mindstudio.ai/blog/typesafe-jev-rlcd-vs-rlhf) (as do I\!) but at the moment we don’t know. In addition to this, it’s highly likely that TypeSafe fine-tuned the model using publicly available decision type datasets (probably including some of the ones I’m using in my build\!). Both the RL and the [fine-tuning](https://en.wikipedia.org/wiki/Fine-tuning_\(deep_learning\)) should make the underlying LLM that they’ve modified produce more accurate answers than otherwise would be the case for decision type questions. My own data shows a nice example of this \- if I change the order of the answers Gemini 2.5 Flash will change its answer about a quarter of the time, Jev on the other hand will do it 9.7% of the time. Assuming they are successful, this training will become TypeSafe’s [moat](https://en.wikipedia.org/wiki/Economic_moat), as they’ll have better real world data than naive implementations. 

So, key takeaways:

- Jev allows a user to specify a piece of input text, which TypeSafe calls the state, and then as many questions as desired.  
- By restricting the softmax function to only the valid options for a given question, Jev can ensure that only valid options are outputted and that the probability distribution adds up to 100%.  
- Jev calculates all the answers to all of the questions about a given piece of input text by looking at the logit scores at the end of each question and ensuring that the question can only see itself, the system prompt, and the initial input text (the state).  
- Because Jev only ever reads the state and the questions once (even when answering a hundred separate questions) it generates all of its answers in the time and cost that a normal LLM would take to generate the first token.

**As ever, all opinions are my own and not those of my employer.**
