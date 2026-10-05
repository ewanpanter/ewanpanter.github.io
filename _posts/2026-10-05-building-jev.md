---
layout: post
title: "Building and fine-tuning my own version of Jev on GCP"
date: 2026-10-05 20:05:00 +0100
---

This is a follow-up to my [first post](https://ewanpanter.github.io/2026-10-05/what-is-jev) on Jev and decision intelligence AI models. When I first read about [Jev](https://typesafe.ai/blog/introducing-system-one-models-and-jev), I figured I could guess how it worked, and probably build my own version that wouldn’t perform terribly. Well this post is me putting my money (about £40 as it turned out\!) where my mouth is and doing just that. I got slightly carried away, so thanks to my wife, Alice, for letting me do this in a week of evenings\!

Note that this post is more techy than my normal ones, but I’ve tried to keep it fairly accessible and explain things as I go.

**Disclaimer: In case it is not obvious, all the below is not associated with my employer, in terms of data, infrastructure, or anything else.**

## My basic plan for SnapCall

Possibly the hardest decision I had was what to call the repo\! Someone had already stolen the best name for a clone of Jev \- [Kev](https://github.com/jaredpalmer/kev), so I came up with SnapCall \- it’s meant to hint at the fact it’s good at snap judgements. With that decided, I sketched out the general gist of what I wanted to do:  

* Try a variety of open-weights models, sizes, and architectures ([MOE, dense](https://developer.nvidia.com/blog/dense-vs-moe-models-active-parameters-throughput-and-when-to-choose-each/), [sliding attention window](https://www.geeksforgeeks.org/computer-vision/sliding-window-attention/)) to see which ones worked best.  
* Fine tune the models using LoRA and see how much improvement is made  
* Build the two Jev tricks discussed in the first post (only output the valid options with probabilities normalised to these, and outputting multiple question answers simultaneously)  
* Deploy on GCP using [GitHub Actions](https://github.com/features/actions) and a CI/CD approach  
* Track and minimise my spend (my offspring are enough of a money sponge without adding another)  
* Demonstrate best practice around tokens and secrets (I will probably make the repo public, so I really, really don’t want to expose my GCP billing account\!)  
* Run the development as a distributed team of [AI developers](https://www.anthropic.com/)  
* Test the accuracy of what I built against the actual Jev model across a range of representative datasets, including some that it probably hasn’t seen in its training data  
* Define a score card in advance to keep me honest

## Model choice and GPU choice

My model choice was largely driven by the cost of GPUs on GCP \- I wanted to use the cheapest one if at all possible. As it turns out, my actual GPU spend was pretty minimal, so I could have used an A100/[H100](https://www.nvidia.com/en-gb/data-center/h100/) without breaking the bank but hindsight is 20:20, so instead I focused on the [L4 GPU](https://www.nvidia.com/en-gb/data-center/l4/). This is a VM that has a Nvidia based 24GB GPU and is pretty [cheap](https://cloud.google.com/products/compute/pricing/accelerator-optimized?hl=en) to use, though you’d still not want to forget you’d provisioned one. 

With the GPU chosen, the memory of the GPU in turn decided the size of model I could use; essentially 4bn and 8bn class models (with a side helping of [26bn 4bit Gemma](https://huggingface.co/google/gemma-4-26B-A4B-it) models). I wanted to try a variety of models and sizes to see what difference it made. I also didn’t want to break any licensing conditions\! A table of the ones I tried, together with their zero-shot accuracy when put into my code and tested on a relevant (to me) [dataset](https://huggingface.co/datasets/Tobi-Bueck/customer-support-tickets):

| Model | Size / type | Zero-shot accuracy |
| :---- | :---- | :---- |
| Jev (reference) | unknown | 0.655 |
| Gemma 4 E4B | 4B, sliding window | 0.656 |
| Phi-4 | 14B dense, 8-bit | 0.629 |
| Qwen3-8B | 8B dense | 0.621 |
| Qwen3-4B | 4B dense | 0.619 |
| Phi-4-mini | 3.8B dense | 0.597 |
| Qwen3-0.6B | 0.6B dense | 0.397 |


 
The sweep showed there was little benefit in increasing the size of the model beyond 4B parameters, so in the end I decided to use the [Qwen3-4B-Instruct-2507](https://huggingface.co/Qwen/Qwen3-4B-Instruct-2507) as my base model. It is a dense model, so the ‘all the questions at once with a mask’ works, which I like and think is [technically](https://www.goodreads.com/quotes/11642962-when-you-see-something-that-is-technically-sweet-you-go) [sweet](http://sweet.It). It will also fit in the memory of the L4 with room to fine-tune and is a relatively new model so has benefited from some degree of [mid-training](https://cameronrwolfe.substack.com/p/midtraining-notes) (I have a theory that the more mid-training a model has, the better its gut instincts).

A few comments on the different architectures:

* Dense (Qwen3 0.6B/4B/8B, Phi-4-mini, Phi-4 14B) \- the masked attention trick works directly, and the newer the model the better the performance.  
* Sliding attention window (Gemma 4 E4B) \- this architecture won’t really work (a question far from the start can’t see the state on local layers) with the masked approach, and needs the cached version. I found it trained about 2x slower.  
* Mixture of Experts (Gemma 4 26B-A4B, 4-bit, on vLLM) \- I only tried this one at the end, so it’s not in this table. The main take away was that MoE actually works pretty well for this approach. If I was going to launch something like this in prod, I’d certainly consider MoE, if for no other reason that there are a lot of good, recently trained, MoE models.

![Full attention vs. sliding window](/assets/images/261005_jev_image12.png)

## My good friend Claude

Whilst I can code, I’m under no illusion that my abilities are now vastly inferior to those of coding agents. As such pretty much all of my coding is done agentically with me checking the outcomes in Github via agent submitted PRs if I’m doing something properly. For this project, I used an instance of a Claude chat to come up with an initial plan \- discussing my objectives and agreeing the best way to set things up. What we jointly came up with in the end was the following:

* There would be a number of workstreams \- each a different instance of Claude Code running on a headless mini-PC that lives in the corner of my home office. Initially there were going to be eight, but we ended up with 11 by the end of the project. Each workstream would have its own session and folders working on its own git branch. The workstreams covered everything from infrastructure to writing documentation.  
* A [CLAUDE.md](http://CLAUDE.md) file set down ground rules for the plan \- e.g. no cloud credentials on the mini-pc, only touch your own folders, don’t push to main, that kind of thing  
* A plan file that was mainly edited by me and the original chat file. It also holds a log of key decisions.   
* There would be a single contract which was a schema that defined every format that the streams shared, only I could approve changes. Allowed things to be developed in parallel without breaking.  
* The agents coordinated via a GitHub issue, which essentially acted as their message board. They shared started/done/failed/decided messages \- for example to queue GPU jobs (since I limited the project quota to one). This worked extremely well.  
* An ‘owner’ inbox where they could ask me to do things and I could reply to say whether I’d done it or not \- this was useful as I could look at the messages and get more context for the PR merging order. Required a bit of fine tuning to stop them bothering me with any random thing.  
* I told them to decide most things between them, unless it cost more than 50p, couldn't be undone, changed a headline, or was one of the things I reserved for myself.

At the end of the development process, they’d made 98 merged PRs. Overall it worked well, I had a single incident of ‘oops I’ve deleted a bunch of stuff’ from one of the agents, but because it’d been following the rules (plus the guardrails I imposed via permissions) it didn’t really matter. As with running a human team, clear ownership, shared documented rules, and a single escalation path worked well.

In case it’s not obvious, one of the things I had them build was all the graphics in these posts. Since I don’t like how Claude writes, I had them build them in an interface, so I can edit the text, colours etc. Here’s an example:

![My team of happy AI minions](/assets/images/261005_jev_image5.png)
<!--more-->
## Building the thing

The first post in this series has the theory of what we’re doing, so I will (try) to keep this brief. To do the actual inference I used [Hugging Face’s transformers library](https://huggingface.co/docs/transformers/en/index), but bypassed its normal loop \- instead I call the model's forward pass directly, apply my own attention mask and position numbers (or a saved copy of the state's KV cache for the sliding attention models), and read the scores at the end of each question. The only exception to this was the 4-bit Gemma 26B model which was run on vLLM as Transformers doesn’t work efficiently with 4bit MOE models (and it would have messed up my timing comparisons). This is a classic example of the power of coding agents \- I discovered that Transformers wasn’t going to play nice, so I asked one of the bots whether vLLM would work in some way…. 10 minutes later it’d rewritten a core part of the project for an entirely different inference engine\!

The Claudes implemented four key bits of functionality for me (background in part one):

- Do everything at once: either via all branches in one token sequence with a mask and position numbers restarting after the state, or via reading the state once and caching with all branches as one batch on top. Verified they were equivalent by trying them on the same model \- the difference was 0.00005 (aka close enough\!).  
- Constrain the answers to only valid options, but only reading the relevant tokens and then softmaxing over these. As well as calculating a probability weighted average for any score questions.  
- Calibrated probabilities. This works like the temperature in a normal LLM generation \- putting T above one pulls the scores together so the model is less sure, below one  pushes them apart so the model is more sure. I used a sample of each dataset with temperatures between 0.2 and 10, and then kept the one that gave the probabilities that best matched the ground truth (via log loss \- a 99% and being wrong is worse than a 60% and being wrong). Whilst this did reduce the calibration error a bit, fine tuning had much greater impact. More useful for models that haven’t been finetuned.
- Option-order averaging. I had a theory that out of the box the model was going to favour certain options (e.g. it might favor A or the first option presented). This piece of functionality let me rotate the option order to see if this is actually true (broadly yes) and calculate an order-consistency score.  


![Option-order averaging](/assets/images/261005_jev_image15.png)

## Fine-tuning with LoRA

Fine-tuning is the process of taking a general purpose model and altering it via additional training with the goal of increasing its performance on a specific type of problem. There are a few ways of doing this \- I used [LoRA](https://en.wikipedia.org/wiki/LoRA_\(machine_learning\)) (Low-rank Adaptation) which is a type of [PEFT](https://huggingface.co/blog/peft) (Parameter-efficient fine-tuning). I chose this mainly because I’d done it before and it’d fit on a single GPU (and if i’m honest because it’s the standard/easiest method and built into the Hugging Face peft library). The other approaches and why I didn’t use them for the record:

- Full fine tuning \- takes ages, fine tunes all 4bn parameters of the model. Also needs about three times as much memory as my GPU has \-a big fat nope.  
- Stuck a classifier head on a normal model \- barely counts as fine tuning, and doesn’t allow it to change question.  
- Various sub-variants of LoRA (e.g. QLoRA) \- would have worked, but normal LoRA was fine.  
- Few-shot prompting \- included for completeness, doesn’t count as fine-tuning in my book. It is my standard approach to making AIs better at a task. Fairly easy to bolt onto SnapCall anyway, as you’d just include some examples with each question.

In a nutshell, what LoRA does is instead of changing all 4 billion parameters in a model, you leave them alone, and train pairs of small add-on matrices beside the matrices that make up each layer (the Qwen model we use as our base model has 36 layers). Each pair of add-on matrices is trained by normal gradient descent and back propagation \- at the end of it you end up with about 33M parameters (0.8% of the total). The ‘low rank’ part of the name comes from how much freedom the add-on matrices have \- in SnapCall’s case it’s 16 \- not in any way tuned, it’s just a standard default.

![I didn't have 65GB - LoRA it is!](/assets/images/261005_jev_image7.png)

Whilst it’s not quite as good as a full fine tune, it’s a lot cheaper and faster. A typical fine tune cost about 60p and took under an hour on our fairly slow GPU. I did nine LoRA runs in total, and it cost me a whopping £4.39. It worked pretty well \- I’ll go through the results properly later in the post \- but the chart below shows the high-level. My Qwen fine-tune comfortably beats Jev, Gemma 4 26bn, and Gemini 3.5 Flash-Lite.

![Performance on consumer complaints](/assets/images/261005_jev_image19.png)

I did a few ablation experiments to see how much work the LoRA was actually doing:

* Fine tuning four different models in the same way, got them all to perform at essentially the same level (Qwen3-4B, Qwen3-8B, Gemma 4 E4B, Phi-4-mini).  
* Performance on datasets that they weren’t fine-tuned on didn’t really change with the exception of when I tried the TeleLogs 5G dataset (more on this below)  
* I tried both shuffled option orders and one option order \- it didn’t meaningfully change the accuracy at all.


For anyone familiar with LoRA, the result is unsurprising \- LoRA produced a decent improvement on the specific types of questions it practiced on, but didn’t really improve ‘answer general questions better’ skill. 

## Deploying onto GCP with GitHub Actions

Before we get into the nitty gritty of how I deployed and updated the models on GCP, it’s worth giving a general overview of the solution. A picture says a thousand words so…

![The general overview of how things work](/assets/images/261005_jev_image3.png)  
As you can see, we had Claudes on the mini-PC pushing changes into GitHub. This triggered a bunch of workflows via GH Actions which did things like standard tests, scan for secrets, etc. I would then review and merge the PRs \- anything that impacted identity management (what the service accounts could / couldn’t do etc \- i.e. the things that could be run and would cost me money\!) could only be applied by me (the Claudes didn’t have a GCP log in, nor did they have the permissions to change the workflows folder in GitHub, etc). The Claudes could propose those IAM type changes, but nothing happened until I ran them by hand in CloudShell.  
![Things I did and didn't allow the Claudes to do](/assets/images/261005_jev_image1.png)

Within GCP all the permanent infrastructure was built by Terraform with GPU jobs being spun up from an Action directly (as they were ephemeral). All the results etc were stored in a bucket which was brought back to the mini-PC using another GH action.

In total there were 18 GitHub Actions workflows, doing everything from the initial tests, through to running the Jev and Gemini baselines. Every push was automatically tested and merging into main would automatically apply the change to the infrastructure (except the IAM parts) via changing Terraform scripts. For anyone unfamiliar with Terraform it’s a standard infrastructure as code approach that works well on GCP \- you describe what you want in text files, Terraform has an existing file of what it’s built, it does a diff and makes API calls to make any required changes.

## Finding a GPU \- harder than you’d think

From a conceptual point of view I know that GPUs are very in demand in the world right now, but to be honest I’d not expected it to impact this project at all. After all, I just wanted a single cheap GPU right? Turns out, the shortage of GPUs is very much a thing\! 

Initially I started by using Vertex AI custom training jobs \- however I found that these would just sit there for ages (literally hours) and then time out. I started off using spot jobs, then on-demand, then tried a second region (initially I was using us-central1 which apparently has the most L4 compute, but has its peak time in UK evening time, I then moved to the europe-west4), then flex-start queue \- none of which normally resulted in a GPU. I think I succeeded once in a couple of days of trying after a couple of hours. The most annoying bit is that it just says pending, and you don’t find out whether you’re going to get one for at least 15 minutes.

I’ve been doing my [Professional Cloud Architect (PCA) training](https://cloud.google.com/learn/certification/cloud-architect) and one of bits I was looking at mentioned that when you provision a plain compute engine VM with a GPU, if it’s not available in a given zone unlike Vertex it will immediately (within 20 seconds) tell you if it can’t be provisioned. So I hatched a plan \- I pulled together a list of all the regions that have L4 compute, and sorted them by the order I thought would be least busy at the time I tend to run. I then got the Claudes to create a Github Action that tries each zone in each of the 17 regions in order until it finds one. This worked a treat \- generally I found that Tokyo tended to be where they ended up. I went from waiting 2 hours, to a median wait of 13s and the very worst of 87s.

You need to pop in a few safeguards (e.g. have janitor workflow that deletes after 2 hours) and I found it easier to build a standard image rather than deploy a container (as otherwise I’d have had egress charges), but consider that a top tip if you need a GPU\!

![This is a much better way of getting a GPU than using Vertex](/assets/images/261005_jev_image9.png)

## Secrets and Tokens

One of the things I was very aware of is that automated scanning of public GitHub repos to find accidentally shared secrets (API keys etc) is [very much a thing](https://www.anthropic.com/threat-intelligence-report-september-2026). Being fairly tight I was quite keen to ensure that I did not become a case study in [such a report](https://www.theregister.com/software/2025/11/10/ai-companies-keep-publishing-private-api-keys-to-github/911473).

To prevent becoming the worst type of statistic, I did a whole bunch of stuff \- possibly overkill given I’ll probably put this code in an entirely fresh repo before I make it public, but better to be sure:

* No Google keys anywhere \- GitHub Actions logs into Google with a one hour token via Workload Identity Federation. So there’s no key file to leak.  
* I authenticated the separate GitHub account that the Claudes logged in as via a shortish duration personal access token; it also doesn’t have access to the workflow directory or admin on the repo.  
* The only secret in the project (a Jev API key with a whopping \$5 of credit on it) was stored in the Google Secret Manager. The Claudes have no access to this and it can only be used via a GH Action that only I can trigger.  
* The eight service accounts on GCP each doing a specific job. The CI ones use custom roles, based upon standard Google ones but without the ability to use setIamPolicy to ensure they can’t give themselves more power.  
* Used [gitleaks](https://github.com/gitleaks/gitleaks) (a tool for finding secrets) to scan every commit and in CI  
* The Claudes have no cloud credentials at all \- they do everything via the GitHub Actions, and a few Actions only work when I start them myself on main. The permissions themselves are only ever changed by me manually doing things in Cloud Shell.  
* None of the workflows can be started by an outsider, they only run on pushes, manual starts or other workflows, all of which need write access to the repo. A classic way to steal secrets is for a random person to submit a PR that triggers a workflow \- I have a test that checks this as well.

Probably not perfect, but combined with short expiry times on tokens and API keys, should be fairly robust. Famous last words, though I’ve already deleted the Jev key, so my remaining \$4.30 is safe\!

## The challenges of data

In order to test and finetune our creation, we need to find some representative datasets. Ideally ones that can have multiple questions asked about each so we can test out this feature. I settled on the following data sets (these are all publicly available):

* Telecoms customer messages ([Bitext](https://huggingface.co/datasets/bitext/Bitext-telco-llm-chatbot-training-dataset)): 26k fake messages to an operator. What is the customer intent  (26 options)? Which category (7)? Are they thinking of leaving (yes/no)?  
* Support tickets ([Tobi-Bueck](https://huggingface.co/datasets/Tobi-Bueck/customer-support-tickets)): fake IT support tickets. Pick the queue (10)? What type (4)? Urgency (low/medium/high)? Is it about billing?  
* Consumer complaints ([CFPB](https://www.consumerfinance.gov/foia-requests/foia-electronic-reading-room/cfpb-consumer-complaint-database-narratives-archive/)): actual complaints sent to the US Consumer Financial Protection Bureau in June and July this year. This was the best large source of real, recent, messy decision text I could find. Which product (11), which issue (26), and did the customer get money back (only 9% did \- guess we should be glad we have the [Consumer Rights Act](https://www.legislation.gov.uk/ukpga/2015/15/contents) I suppose\!)? This was my primary test.  
* 5G network faults ([GSMA TeleLogs](https://huggingface.co/datasets/GSMA/ot-full)): test data from 5G networks, with tables of signal, speed and throughput. Trying to assess root cause for drop in throughput. Pretty hard and probably needs reasoning.  
* Security advisories ([Red Hat CVEs](https://access.redhat.com/security/data)): vulnerability write-ups from April to September 2026, that I had AI strip the severity out of the text. How severe is it (4 levels), and where can an attacker exploit it from (4 options)?  
* A set of completely unrelated question set to find out if doing the fine tuning destroyed normal performance. A "did I break it?" set ([AG News](https://huggingface.co/datasets/sh0416/ag_news) and [BoolQ](https://huggingface.co/datasets/google/boolq)). 

I also removed near duplicates so that the sets weren’t full of near repeats which would probably skew the results. Each data set was split three ways \- the majority for fine-tuning, a tuning data set, and then a test set that was withheld until final scoring. For CFPB that was a 3000 / 300 / 1000 set.   
![Examples from each dataset](/assets/images/261005_jev_image10.png)

The test set I trust the most is the CFPB set \- for two reasons:

- It’s a real data set, some of the others (e.g. the telecoms customer messages and support tickets) are wholly synthetic so are cleaner and more formulaic than real life examples.  
- Pretty much all data on the internet is in the training dataset for LLMs by this point, especially anything hosted on Hugging Face. As a result I’d expect Jev to have seen this data before (either because it was finetuned on it, or because the underlying model has seen it). I’ve **tried** to minimise the impact of this by making the CFPB data my main test \- this was only put on the internet in the past couple of months, so it won’t be in underlying models data, and there is a solid chance that Jev never saw it in training either (depending on the training data cut off).

## Ta-da\! The results

So, how did SnapCall do? First let’s look at the zero shot models vs Jev. 

![Speed and accuracy for CFPB](/assets/images/261005_jev_image4.png)

Accuracy wise as we can see, it’s a bit of a wash. No-one wins big, we have our base 4B model at six points behind Jev, Gemma E4B two points behind, the 26B MoE model level, and Flash-Lite 4 ahead. I think the 26B results is the most interesting here \- it is dead level with no post training at all, at 214ms \- you can self host a Jev-level model on your laptop if you want (I’ve had the 26B model running on my 24GB Macbook with no problems at all \- will be a bit slower than the L4 I expect). One interesting result is that on the ‘Relief’ (did the  customer get a refund) question Jev is only a bit better than a coin flip \- worth noting that this category is very lopsided, answering no every time would get you 91% hence using AUC. For reference, answering the most common answer to every question would score 0.44. 

Timing wise we can see that Jev is ahead \- by a shade under 40ms over my best result. This is actually a pretty good result as although Jev includes network latency, they certainly aren’t running it on a single L4 on unoptimised code\! I’d expect a very meaningful speed up on a H100 but I’ve not measured it \- if I was going to take a [SWAG](https://en.wikipedia.org/wiki/Scientific_wild-ass_guess) I’d say maybe 15-35ms per complaint for the 4B Qwen and maybe 45ms for the 26B Gemma model. The take away being that the models are extremely comparable to Jev’s performance, and all of them being vastly faster than Gemini even in its quick Flash-Lite guise.

One important result not shown in the graph is what happens if you shuffle the order of the options. Here Jev has a clear advantage over the other models \- when the options were shuffled, Jev only changed its answer 9.7% of the time. Gemini 2.5 Flash (I didn’t test 3.5 Flash-Lite) on this changed 26% of the time, Gemma 4 E4B 20%, and Qwen 28.5%. 

Let’s move onto the results when we fine tuned the Qwen3 4B, first looking at the CFPB complaints as above.

![Before and after fine-tuning](/assets/images/261005_jev_image13.png)  
As can be seen, a [Max Verstappen-esque](https://www.youtube.com/watch?v=ZtiQk-vqmBA) performance from the fine-tuned SnapCall. Overall a jump of 18 percentage points to put it 12 points ahead of Jev. The biggest jump is in the Issue category \- this is a result of the model learning how CFPB files complaints. Unlike the base model, the fine-tuned SnapCall is now as steady in its answers as Jev \- flipping based on the option order 9.9% of the time vs. 9.7% of the time for Jev.

Fine-tuning also made SnapCall’s probabilities honest. Expected Calibration Error (ECE) groups the answers by how confident the model was, and compares that with how often it actually got the correct answer \- with a score of 0 being perfect. On CFPB complaints the fine-tuned SnapCall had an ECE of 0.018, so within two points of reality. Jev on the other hand was quite over confident with an ECE of 0.18. Worth noting a fairness point (though I think this is representative of what you’d do in an actual rollout) \- Qwen’s 0.018 includes a temperature fitted on the 300-complaint validation CFPB data, whereas Jev’s 0.18 is as it comes. With no temperature fitted for either model, it’s 0.024 against 0.18.

There is another related trick that can be done because every answer comes with a probability: if you order your test set by winning probability along with the actual answer, you can pick a threshold probability and calculate what share of the answers above this threshold are correct. E.g. you could say something like, if you only take answers where SnapCall is at least 70% sure, then those answers are correct 90% of the time on average. This is a useful thing to do, as it can be a good signal to let you know when you need to hand off to a human (or a smarter AI). If you actually work this out, you get the following chart. Couple of things to note \- Jev never quite makes our 90% accuracy, and the Gemini probabilities come from asking it to include them in its output (you no longer get them in the API response for newer Flash models).

![When to pass to a human](/assets/images/261005_jev_image20.png)

If we look at overall performance across all the datasets we can see a similar story (note, if I’m being fully strict, there is one issue here with these test sets, Qwen and Jev were scored by asking each question in four option orders and averaging, while Gemini was asked in a single order. On CFPB, where I did both approaches, this makes no difference for Jev or the fine-tuned model (under 0.2 points), and only flatters the untuned Qwen. All time numbers below are for a single call) :

![Results from the other datasets](/assets/images/261005_jev_image16.png)  
Again the fine-tuned SnapCall surpasses Jev on everything except the TeleLogs (which they all did poorly in), though it's worth pointing out that this is all a bit unfair as Jev wasn’t specifically fine-tuned (probably) on this data. It does underline the point that a little cheap fine-tuning goes a long way if you know the type of question you are going to be asking.

Now let’s talk about what didn’t work. Mainly the 5G network faults test set \- this is what I expected. Qwen performed (fine-tuned or not) at a level equal with chance. Jev was a bit better, but it was still correct only 36% of the time. If you drill down into this, it got the causes correct where one number exceeded some threshold, it didn’t get it right where it had to do any maths or consider geometry. The Qwen fine-tune for TeleLogs has a near perfect ECE (0.005), because it learnt to say about 12.5% for everything (i.e. random chance) \- honest, but useless\! The take away is that this is very much Decision Intelligence doing what it says on the tin \- system one thinking, rather than system two.

The security fine-tune did improve the performance significantly of SnapCall but caused it to narrowly fail the forgetting check. Likewise the TeleLogs dataset wrecked the model on the out of domain test. Worth bearing in mind that this is always worth checking in a production use case.

## What I’ve learnt and take-aways

All in all the whole build came to about £38 against my self-imposed budget of £30ish. The breakdown was roughly (excluding some bits and pieces):

* £18 \- GPU time. This was across dozens of separate jobs. All the fine-tuning came in at a shade under £4.50, with the CFPB one that beat Jev at 62p. Higher than it had to be, as I often ended up using Tokyo that has up to a 25% premium over the US (probably why it had free GPUs\!).  
* £18 \- Gemini. This includes a mistake that cost a fiver. Still, given the relative number of runs I did on Gemini vs everything else,  it really underlines the point that these types of models offer substantial economic benefits.  
* \$0.70 \- Jev \- it really is extremely cheap\!

The model fine-tuning work on the GPU was the cheap bit. I haven’t included the cost of the Claudes as this was all done on my normal plan. 

One thing that is useful is to look at the interaction between accuracy and cost. Here’s a picture that tries to do just that:

![The pareto-frontiers for cost and time](/assets/images/261005_jev_image18.png)  
There is a lot going on in these charts, but the thing to know is that a) it’s a log scale and b) you want to be in the top left.

The left-hand chart is showing accuracy vs. speed with the right showing accuracy vs. cost. The dotted line and solid line are the [pareto-frontiers](https://en.wikipedia.org/wiki/Pareto_frontier) for each \- the best you can get at a given speed or cost. Without fine-tuning your options are Jev (fast) or Gemini 3.5 Flash-Lite (a bit more accurate but 16 times slower and 40 times more expensive). If you put in the fine-tuned model, it dominates on cost and is only just behind Jev on speed \- especially impressive if you take into account the lower performance hardware it’s running on. 

However, there is a big catch \- the numbers in the graph assume the GPU is busy all the time. An L4 costs about £13 a day, and you pay that whether it’s at 100% utilisation or 10%. What this means is that you need to smash the best part of a million decisions a day into it in order to outperform Jev on a cost basis. And that’s ignoring the enterprise realities of dev and prod environments, fail-over backup, management overhead, etc.

So, if cost is the only thing that matters, then you should go and use Jev. But there are lots of reasons why you might not want to for example:

- If you’re a telco and you want to use it on your core network, you’ll be wanting to have a deep and meaningful chat with whoever knows most about the [Telecoms (Security) Act](https://www.legislation.gov.uk/ukpga/2021/31/contents) before you start sending data off to a new start-up that [processes your data in the US](https://typesafe.ai/legal/privacy-policy). The same extends to any enterprise that has [GDPR](https://gdpr-info.eu/) or [sovereign AI](https://www.mckinsey.com/featured-insights/mckinsey-explainers/what-is-sovereign-ai) constraints.  
- As I’ve demonstrated you can achieve a meaningfully improved performance if you’re prepared to collect some data and do some fine-tuning. It’s quite likely that the data will already exist in system logs so this is probably a low barrier (e.g. if you have a person allocating tickets to queues).  
- The realities of enterprise vendor management often makes certifying a new model provider that came out of stealth mode less than a month ago slow and painful. That being said, more established vendors (e.g. OpenAI) are now offering their [own equivalent service](https://decisionapi.net/).

The implementation I’ve demonstrated here is very much an evaluation oriented set up \- it is not designed to serve live requests. Realistically you’re not going to be using Hugging Face Transformers to serve this, you’re going to customise [vLLM](https://vllm.ai/). This is relatively straightforward (I’ve shown with the Gemma 4 26B model that both of Jev’s tricks work on vLLM), but it’s still a substantial engineering effort. Additionally, you’d want to do things like sending items below a confidence threshold to a person, integrate with other applications, continually capture live traffic to catch model drift, and probably serve on [Cloud Run](https://cloud.google.com/run?hl=en).

So what are the take-aways from the rabbit hole I’ve taken myself down?

1. **Jev is really quite good.** It’s fast (around 100ms), extremely cheap (4p for 1000 complaints), and steadier than any of the zero-shot models when the options are shuffled. If you have no labelled data and your questions fit into the system one thinking mould then it’s a decent default (noting my points about GDPR etc above).  
2. **With labelled data you should roll your own and get better performance**. With a few thousand labelled examples I’ve shown that you can beat Jev by 12 percentage points with a spend of 62p and an open-weights model. It will run at a comparable speed to Jev and its confidence scores are honest enough to be used to judge when you need to pass to a real person.  
3. **Fine-tuning doesn’t generalise**. As expected, the fine-tuning fit the model to the task, it didn’t make it better at all decision datasets. It also managed to wreck the general ability for one dataset, so testing and knowing if you’re likely to see out of domain inputs is critical.  
4. **System two tasks don’t work**. Very much a case of [does what it says on the tin](https://www.ronseal.com/the-ronseal-brand/the-ronseal-phrase/), but my results underline this \- if your decision involves reading tables, doing sums, or detailed reasoning, Jev and similar are not a cheat code.  
5. **You can get zero-shot Jev level performance on a laptop**. If you’re prepared to increase the [size of the model to 26B](https://huggingface.co/google/gemma-4-26B-A4B-it) and wait 200ms instead of 100ms then you can get Jev level performance without even finetuning. On a model you can run (somewhat more slowly) on a MacBook.  
6. **Running a team of AI developers works if you provide rules and structure.** Give some AIs clear ownership, a single shared contract, a CI/CD pipeline, and a message board, and you’ll find they’re extremely effective. Not unlike what you need for a human team. [I for one welcome our new beige AI coding overlords](https://www.youtube.com/watch?v=8lcUHQYhPTE&xstg=CAMSEBUJ_b-oH-PhF0yjBgaukzY%3D).

Finally, TypeSafe's marketing talks a lot about RLCD (Reinforcement Learning for Calibrated Decisions). This is a phrase that they came up with, and they’ve not explained how it works or what it means \- so I can’t explain what it means. I can say what I measured though \- when you shuffle the options on a question Jev is noticeably more steady than non-fine-tuned models. This suggests it has been trained for this job, however with the data I used, its probabilities were not well calibrated. On the CFPB complaints its calibration error was 0.18 (over confident by 18 percentage points) against my 0.02 for my 62p fine-tune. There may be workloads where it does deliver better calibration, but you’ll certainly want to check to see if that data includes your data. 

## Can I have the code

Sure \- but I need to duplicate the code into a new repo and be absolutely sure I’ve not left anything in there that will cost me money\! So, give me a few days.

**As ever, all opinions are my own and not those of my employer.**  
