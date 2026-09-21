---
layout: post
title: "The governance bill for cheap AI safeguards"
---
In my [last post](https://ewanpanter.github.io/2026-08-24/How-safeguards-became-almost-free-(sort-of)) I showed how using simple linear probe classifiers combined with a small LLM as a judge has transformed the cost profile of AI model safeguards. In this follow-up post I will demonstrate the potential for misuse. This almost entirely derives from the extremely low cost of running thousands of probes on every single conversation.  

With a practically unlimited ability to sample the meaning / topic of a conversation without [incurring additional compute cost](https://arxiv.org/abs/2601.04603), it becomes technically possible for a model provider to perform speculative searches. Whilst the value of any one probe trigger in a large population may be low (as even a low false positive rate in a sample of billions is still a [large number](https://en.wikipedia.org/wiki/Law_of_truly_large_numbers)) there are plenty of cases where patterns of triggers can be useful in ways many people and enterprises would not love.

## Silent steering

For an enterprise perhaps the most obvious threat model is from silent steering of the model output based on detection of a specific type of input. A fantastic example of the entire threat model was provided by Anthropic themselves when they launched Fable 5. When I read the [system card](https://www-cdn.anthropic.com/d00db56fa754a1b115b6dd7cb2e3c342ee809620.pdf), I, like many others, was pretty surprised to come across this sentence on page 13 (red boxes are my own):

![Extract from the Fable 5 system card](/assets/images/260921_safeguards_image2.png)

The plain English translation of this is that if Anthropic don’t like what you are doing with their model, then they will manipulate the output without informing the user. In this example they were deploying this capability against anyone they believed to be using the model to develop a competing frontier model.

To be clear, Anthropic frames this as a [safeguard](https://www.anthropic.com/threat-intelligence-report-september-2026#illicit-distillation-sep-26) in the context of recent advances in model capability to [accelerate AI development](https://openai.com/index/an-alien-mind/). However, it is not illegal to develop a frontier LLM. In its least charitable reading, this is a commercial provider changing the output the user is paying for to suit their commercial interests without revealing to the user that they have done so. 

There was a lot of [press](https://www.wired.com/story/anthropic-walks-back-policy-fable-5-ai-researchers/) [attention](https://www.engadget.com/2192004/anthropic-walks-back-policy-sabotaging-research/) about this part of the system card. Anthropic pretty quickly [walked this back](https://x.com/claudedevs/status/2064949876463645026) and removed this silent steering approach. It is to their credit that they have left the text in the system card and for what it’s worth I tend to believe that this was intended to be exclusively a safety mechanism.

In case it’s not obvious why this is problematic, let's demonstrate it with the proof of concept I built (see note at bottom for how I built this). Here a linear probe and LLM cascade identify when a user is discussing a hypothetical company named ‘MobileCo’. If it triggers then the prompt is silently modified to ensure that the result is favourable to MobileCo. 

**Note:** I don’t think any telecoms company is doing this - I only used telecoms for the example as that’s what my probe was already trained on from post one. 

![An example of silent steering that is invisible to the user](/assets/images/260921_safeguards_image1.png)

As you can see the user asked for an honest view on coverage vs competitors, which the model was going to give to them, until the prompt was silently rewritten to ensure that the response said MobileCo was superior. This is a simplistic example and you could achieve much the same thing just by editing a system prompt - however probes would allow a more sophisticated approach with potentially quite subtle topics becoming targets for manipulation without the user being aware (and potentially the company providing the chat assistant if they are using a third-party LLM provider).

Another good example is using a code detection probe to trigger the silent rewriting of a prompt to include an [exploit](https://en.wikipedia.org/wiki/Exploit_(computer_security)) that writes [PII](https://en.wikipedia.org/wiki/Personal_data) to a ‘log file’.

![Detection of a code type and insertion of exploit](/assets/images/260921_safeguards_image3.png)

Again, the proof of concept works as expected, and again this is more simplistic than a real attack (somewhat constrained by this being a [4bn-parameter model](https://ai.google.dev/gemma/docs/core)!). In reality, if you were a compromised or malicious model provider you’d probably have several probes to help pinpoint the target (MobileCo AND C++ AND Salesforce etc) from a large volume of data as well as use a more subtle exploit technique like steering the model activations towards producing vulnerable code rather than some explicit exploit.

Even without a bad actor trying to inject insecure code into your prompt responses, it should be obvious that silent steering is [not something to be welcomed](https://jonready.com/blog/posts/claude-fable5-is-allowed-to-sabotage-your-app-if-youre-a-competitor.html). Any classifier, along with its associated judge LLM, is going to be imperfect and as such will generate a degree of false positives (and negatives). Let’s take Anthropic’s implementation as an example - let’s say you’re not trying to train a frontier model, but for whatever reason the safety guardrails decide that you are. The end result is that you’re going to get an inaccurate answer from the model for whatever you’re trying to do with no way to know that it’s wrong. Not ideal.

## Cold-start targeting

Linear probes allow population-wide, continuous, multi-dimensional targeting at inference time. A bad actor can maintain hundreds or thousands of classifiers over every prompt that is submitted - “involves company X”, “works in finance”, “deals with SAP payment modules”, “drafts emails”, etc. Triggering an individual probe means little, but the cumulative triggering of probes over multiple conversations provides a very powerful signal that the user is worth targeting. To make this concrete:

* **Stage 1: Ubiquitous surveillance.** A bad actor applies linear probes continuously to score every interaction of every user for concepts of interest.  
* **Stage 2: Cumulative scoring.** Triggers or probe activation scores amass gradually over time to build a profile of each user.  
* **Stage 3: Escalation to LLM.** Promising profiles are escalated to an (expensive) LLM analysis of triggering conversation history, and information of interest is extracted.  
* **Stage 4: Exploitation of data.** An agentic system generates and executes [highly personalised phishing](https://www.kaspersky.com/resource-center/definitions/spear-phishing) or other attacks on the user or their colleagues.

The key thing to note about the above is that the user was identified by the concepts they discussed rather than being deliberately targeted through their identity as in traditional spear-phishing attacks. I’ve called this cold-start targeting, as an analogy to cold calling - the difference is that this is much more likely to be successful!<!--more-->

The entire attack is likely end-to-end automatable with existing [open-weight models](https://artificialanalysis.ai/agents/coding-agents), and this will certainly be so in [6-12 months](https://www.aisi.gov.uk/blog/how-far-behind-the-frontier-are-leading-open-weight-models-on-cyber) when these models approach the capabilities of Mythos / Fable-class frontier models. The only thing needed is model activations that can be accessed by a bad actor - that might be through a provider compromise, or because the model provider is themselves a bad actor. 

A possible pushback against this attack is that most employees have good AI hygiene and don’t put sensitive information into non-corporate AI models. Even if we pretend that [shadow AI](https://www.ncsc.gov.uk/blogs/the-hidden-risks-of-shadow-ai) is not a thing and that all employees follow corporate AI policies, this type of attack can succeed even where the employee has been relatively careful with what they put in individual conversations as the triggers accumulate across conversations. Individually innocuous disclosures can be dangerous when looked at as a [body of information across their entire history](https://en.wikipedia.org/wiki/Mosaic_effect).

It’s reasonable to object that this is nothing new - after all, technically this attack is possible without linear probes. A bad actor with an unlimited budget could run a honeypot AI service and scan every interaction with an LLM for information. What probes transform are the economics - it essentially costs nothing to identify the best targets across a very large population, meaning you only use the LLM analysis where the ROI justifies it. 

## New metadata = new risks

One of the slightly more concrete things that comes from the proliferation of linear probes is associated [metadata](https://en.wikipedia.org/wiki/Metadata). Every time the probe classifies a prompt, it creates a new bit of data that states whether or not the probe thought the prompt was the thing it was attempting to classify (in actuality, probably the probe [confidence score](https://en.wikipedia.org/wiki/Scoring_rule) rather than a binary yes/no). Depending on what the probe is classifying for, this is pretty useful information. It is also not obviously going to be classified by the model provider as customer data. 

You can imagine scenarios where a provider decides they want to know what type of work their customers are doing with models to allow them to better direct their product development - some carefully targeted probes would likely make this pretty cheap to do at scale (compared with the alternative of sampling prompts, summarising with an LLM, etc). Assuming your probe was reasonably accurate, you’d never even need to do the LLM as judge stage - you’re not particularly bothered by false positives/negatives at a thematic level.

The above is the benign version. Given you can probe on any concept, it’s pretty straightforward to imagine versions of this that move from fine (safety classifiers) to grey (product improvement probes) to entirely uncool (market-moving information). 

You may think that is not a realistic threat and existing enterprise agreements rule this out - but actually the picture is not straightforward. For example, in a key licensing document ([Microsoft Products and Services Data Protection Addendum (WW)](https://www.microsoft.com/licensing/docs/view/Microsoft-Products-and-Services-Data-Protection-Addendum-DPA)) Microsoft are granted permission to create information from logs, but  “in each case without accessing or analyzing the content of Customer Data or Professional Services Data” - this rules out this technique. 

On the other hand, [OpenAI](https://cdn.openai.com/osa/openai-services-agreement.pdf) states that “OpenAI will not use Customer Content to develop or improve the Services, unless Customer explicitly agrees to such use”. Which seems fine, but then the definition section at the end states that “‘Customer Content’ means the Input and the Output”. As I read it, the implication is that derived data is out of scope of the clause - including metadata derived from the customer content itself.

[OpenRouter](https://openrouter.ai/about) is an example of a provider who specifically calls out their storage of content metadata - their [terms](https://openrouter.ai/terms) state that “OpenRouter uses a hosted model for categorizing Inputs, which does not store or log any Inputs provided to it”. They do go on to say that the categories are not associated with the user or organisation but they retain the right to create [derivative works](https://openrouter.ai/rankings#task-spend) based on the inputs (in other words the categories are retained and used for their purposes - for example on their ranking page).

I’m not particularly picking on Microsoft, OpenAI, or OpenRouter - they’re just the ones I decided to have a quick look at - fairly randomly. I’m also definitely not a lawyer; this is just my understanding of how they read in September 2026 (aka this blog post is not legal advice!). Disclaimers aside, the point is not that one of these three companies is better or worse than the others, it’s that with the proliferation of linear probes, if you’re not squinting at clauses with this in the back of your mind, perhaps you should be.

## Where does this leave enterprise governance?

You’ll note I've not covered a few of the more obvious societal issues - there are some unpleasant things you can do with this technology if you’re an authoritarian government for example. At some future point, I may write a third post on these (don’t hold your breath though!).

Since to a first approximation [all](https://arxiv.org/pdf/2601.11516) the [model providers](https://deploymentsafety.openai.com/gpt-6-astra/protocolqa-open-ended) are going to adopt these technologies, it is not the case that their use can be avoided for enterprise AI workloads (nor should it be really - this approach brings [substantial safeguarding benefits](https://research.google/pubs/aase-activation-based-ai-safety-enforcement-via-lightweight-probes/)). So where does this leave enterprises, and specifically their governance functions?

The answer isn’t clear-cut, although there are some straightforward things to be done. Firstly, in case it’s not obvious, don’t use less reputable providers of inference - you need proper data retention policies, a contracting entity, etc. For your reputable suppliers, consider asking questions along the lines of the following:

* What classifiers or probes run on our inputs and outputs and for what purposes?  
* Are the classification metadata retained, and are they linked to account or user data?  
* Does ‘Customer Data’ include information that is derived from our content?  
* Do usage data or service data clauses cover semantic labels/metadata or only operational information such as latency, tokens, etc.?  
* Are any classifier outputs shared with third parties, and if so, for what purposes?  
* Can you provide a list of active classifiers, and give notice when new ones are added?  
* Do you rate-limit, alter routing, or modify model outputs based on their classification?

Many enterprises host their own LLMs - it would be worth ensuring that there are robust and auditable answers to the above questions (even if only offered to your employees).

Obviously, there are [harder questions](https://arxiv.org/pdf/1804.08730) to address. 

For example, you can only ask the questions above if you have a contractual agreement with the company concerned. Shadow AI is very much a thing, and you’ve not got a relationship with the people providing these services to your employees. The solution is probably not a crackdown - it’s to address why employees are using such services in the first place. If your corporate-sanctioned AI is a) available and b) on a par with frontier services, then the problem largely goes away without hard-to-enforce restrictions.

Similarly, assurance of new models is now harder. Previously when approving a new model for corporate use you could run your [tests and evaluations](https://inspect.aisi.org.uk/) and be reasonably sure that everything was going to be good until the next model release. Probes give a very low cost way of silently varying the model output based on a probabilistic assessment of a prompt.

Two solutions leap to mind: contractual commitments around modifying outputs and treating testing and evaluation as a continual process rather than a one-off event. The potentially significant additional costs of the latter can be mitigated by automating the assessment process. This comes with other benefits, for example, model providers often do minor [post-release tweaks](https://arxiv.org/abs/2307.09009) that could in theory impact results - this approach would highlight any issues.

Overall, safety classification through linear probes is undoubtedly a good thing - I hope it’s clear that the attack scenarios are hypothetical worst cases rather than things I’ve seen in the wild. Linear probes allow safeguarding at a [much larger scale](https://alignment.anthropic.com/2025/cheap-monitors/) than previous methodologies and direct the bulk of safety compute resources at actually problematic inputs. That being said, it is this low cost that moves the subject from safety into governance - traditionally, metadata meant throughput, token use, latency. It doesn’t anymore.

**As ever, all opinions are my own and not those of my employer.**

*(From my first post: a quick note on what I built. Essentially the demo is a Streamlit front end over a pipeline that orchestrates multiple probes, associated judges, and then an intervention layer. Each probe is a linear classifier trained on the optimal layer or layers (determined in training via a layer sweep). The tokens are each evaluated against a moving average threshold to ensure no rogue triggers. If a probe is triggered then the message (or entire exchange depending on the settings) is escalated to a judge (one per probe, so if multiple probes are triggered, multiple judges will evaluate) - the verdict determines the action (allow, refuse, silently rewrite). Behind it sits a swappable runner that lets me change the model with a config change. Training the probes was a standalone exercise: generating balanced sentence pairs, capturing activations, and a fair bit of iteration to avoid creating a keyword detector. A good few weekends and evenings of work!)*
