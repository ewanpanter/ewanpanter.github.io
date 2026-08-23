The economics and approaches used for AI safeguards have recently changed quite dramatically, and this has largely passed by unnoticed.

I find this kind of thing interesting and recently gave a couple of talks at work about how safeguards work in frontier AI systems, and demonstrated how I implemented one of the key new techniques on a 4bn open weights model. This approach has dramatically reduced the cost of running safeguard systems, but also opened up a number of new threats. I thought it’d make an interesting set of posts for my blog - so here we are.

This first post will provide an overview of how modern safeguards work. The next post will concentrate on some of the risks that enterprises now need to consider. I’ll then cover some of the wider societal issues in a third post, and finally the fourth post will be a reasonably geeky write up of how I actually implemented linear classifiers on an open weights model.

(I’d put the chances of the first two seeing the light of day as at least 90%, on the grounds that a) you’re reading one of them and b) I’ve written the other one. The last two are somewhat more embryonic - but hey the nights are drawing in!)

## What is a safeguard

Safeguards, or guard rails, are a key component to any commercial deployment of AI. In a nutshell they prevent a system from being used for a purpose that the provider would rather not have their system being used for. When you ask a customer service chatbot to write some code for you and it refuses - that’s a safeguard ([or not](https://programmerhumor.io/python-memes/chipotle-support-bot-solves-linked-list-now-a9nz)). If you ask ChatGPT to tell you how to do something illegal and it refuses, that is again a safeguard mechanism kicking in.

There are three basic things you want from your safeguards:

1. To be able to tell the difference between a good and bad user prompt

2. To be able to tell the difference between a good and bad model response

3. To be cheap and fast

Obviously no safeguard is going to be perfect and it’s a provider call on whether they want to err on the side of false positives and risk annoying their users, or false negatives and risk the model doing [something naughty](https://huggingface.co/blog/security-incident-july-2026).

## Why safeguards look like swiss cheese

There is no perfect safeguard. No [silver bullets exist](https://www.imdb.com/title/tt0280609/) unfortunately. Despite what the [White House](https://www.wired.com/story/the-white-house-wants-anthropic-to-block-all-jailbreaks-that-may-not-be-possible/) might like. As a result frontier safeguards operate on the principle of [defence in depth](https://en.wikipedia.org/wiki/Defense_in_depth_(computing)).

This is a bit like a stack of swiss cheese - if you take any individual piece (an individual safeguard) you can look through the holes - but if you stack enough bits of cheese on top of each other the holes don’t line up perfectly and you can’t see through the stack at all.

![A diagram showing the safeguard stack](/assets/images/260824_safeguards_image14.png)

The diagram above is my noddy attempt at trying to show the major components of a modern safeguarding stack (yes, yes you can be pedantic and argue that the system prompt and model training aren’t layers in the same sense, but whatever). Let’s discuss them in order:

- Monitoring and trad IT - So the furthest out defence is what I’ll call trad IT. e.g. blocking known dodgy users / IPs, monitoring suspicious patterns - if someone has just submitted 5k coding textbook questions to your end point, they’re probably [distilling your model](https://www.anthropic.com/news/detecting-and-preventing-distillation-attacks). Likewise if someone is spamming weird bits of text in base64 you may want to investigate whether they’re [jail breaking you](https://time.com/collections/time100-ai-2025/7305870/pliny-the-liberator/).

- Next you have your first bit that’s specific to the LLM - the input classifiers. These do what they say on the tin, they classify the input prompt as either pass or fail. You normally have a bunch here - one for jailbreaks, some for forbidden topics, does it have PII in it etc.

- The heaviest lifting is largely done by the model itself via its [alignment training](https://alignment.anthropic.com/). Essentially you train it to avoid talking about illegal stuff or whatever. Typically done with RLHF and training on examples of models refusing bad prompts.

- The penultimate defence is the system prompt. Even if you specify your own system prompt via API, there is often some kind of [internal prompt](https://model-spec.openai.com/2025-04-11.html) that can’t be turned off. The system prompt will give the model some basic rules and information, including how it should talk and what things it should not discuss.

- The final bastion is the output classifier. This is largely for the extra cunning attacks where the input is benign but the output it generates is not. For example a [multi-turn jailbreak](https://www.promptfoo.dev/docs/red-team/strategies/multi-turn/).

The only other defences to mention are the offline safeguarding activities - all the frontier companies do [red teaming internally](https://openai.com/index/unlocking-self-improvement-gpt-red/), and with external partners like the [UK AISI](https://www.aisi.gov.uk/). Additionally, since 2024 the US government has been releasing quite significant security ‘guidelines’ to datacentres housing [frontier models](https://media.defense.gov/2024/apr/15/2003439257/-1/-1/0/csi-deploying-ai-systems-securely.pdf).
<!--more-->
## How do traditional input / output classifiers work?

The rest of this post is going to concentrate on the input and output classifiers - the bits that actively decide if the input or the output is naughty.

Traditionally these are implemented through a string of models. A picture says 1000 words, so here is a diagram [I drew](https://excalidraw.com/) to demonstrate the principle:

![A serial string of classifiers and models](/assets/images/260824_safeguards_image5.png)

As you can see, before your prompt ever sees the big model it’s gone through a smaller LLM which reads the input and decides if it’s suss. Typically this is the model provider’s smallest LLM where they’ve fiddled with the final layer to output a pass / fail instead of words - for example Anthropic use a [fine tuned version of Haiku](https://arxiv.org/pdf/2501.18837). The output classifier uses the same approach - a small model that reads the output and decides pass or fail.

This is probably an oversimplification. In reality there are probably several classifiers executing in parallel rather than one model with a very long list of rules (several small checking prompts will be quicker than one very long one - plus keeping the rules simple increases accuracy with small models). Additionally it’s likely the prompt is actually passed straight to the main LLM whilst the input classifier is still deciding its answer - again in the interests of speed.

Corporate IT systems will often also implement their own [internal AI classifiers](https://www.mobileeurope.co.uk/vodafone-doesnt-discount-moonshots-but-looks-to-ai-milestones/) which will generally work in a similar fashion.

However implemented, this system does one thing very well - **it introduces cost and latency**. For every prompt submitted to an LLM there are actually (at least) three model calls being made - two to the input / output classifiers and one to the actual big model. From a user point of view, only one of those is adding value.

This then drives a further issue, in principle there is no reason not to pass the entire chat history to your classifiers as this will increase the accuracy as it’ll provide greater context. For example this is important when you’re dealing with multi-turn jailbreak attacks. However in practice, no one does it as instead of passing maybe 1000 tokens to the model, you’re potentially passing hundreds of thousands of tokens - more than likely cost and latency prohibitive.

The astute of you will have noted I’ve titled this ‘traditional’ - and yes there is a new, vastly better approach that is now being adopted across the industry.

## Constitutional Classifiers ++

I think we can start this section by admiring what a truly terrible name has been chosen for this approach by [Anthropic](https://www.anthropic.com/research/next-generation-constitutional-classifiers). Even in an industry where GPT5.6 Sol is seen as a reasonable product name, it’s pretty special.

With that out of the way we can admire what is genuinely a [technically sweet](https://www.oxfordreference.com/display/10.1093/acref/9780191826719.001.0001/q-oro-ed4-00007996) solution to the cost and latency issues of the traditional approach, whilst also allowing a much more robust implementation. Once again a picture is probably the easiest way to explain this, so [here’s one I made earlier](https://www.youtube.com/watch?v=2UzWkEHOdZA):

![A comparison of the old approach and the linear classifier based approach](/assets/images/260824_safeguards_image15.png)

The basic concept to understand is that rather than sending every prompt to a classifier LLM, you can instead read the main model’s internal activity with many tiny, close to free probes, that you can scale to look at as many things as you like. Only if one of those very cheap and fast probes triggers do you pass the message to a small LLM to do a more detailed check.

The analogy is a smoke alarm - it’s always on, it basically costs nothing to run, and only if you hear the alarm do you bother to go and see if it’s the toast burning or if someone is torching the kitchen.

The technical sweetness is that the probes work by checking numbers that have to be generated in any event as part of the answer generation. There is no real incremental work to be done at all other than a quick matrix multiplication (something that GPUs are astoundingly good at). Only if something potentially sketchy is found then does incremental work happen.

This saves money. A lot of money. According to [Anthropic](https://arxiv.org/pdf/2601.04603) running this system is 28.5 times cheaper to run than the traditional dual classifier approach (Anthropic also describe it as a 1% overhead vs running the model with no safeguards at all). The saving largely comes because they’re now only sending 5% of the traffic to the smaller LLM - they could actually have saved more here, but they’re spending some of the saving sending the entire conversation to the smaller LLM when the probe is triggered.

In addition to the cost savings, the approach is considerably more robust with a 3.7x reduction in the jailbreak discovery rate and a zero universal jailbreak count - even after 1,700 hours of red teaming (their previous implementation had four universal jailbreaks found by redteamers each taking only 27 hours to find on average - as far as I’m aware there are still none after months). Due to the greater context the small LLM now has, there has also been a ~7.6x reduction in improperly refused or flagged interactions.

The benefits are such that it is pretty much a no brainer that every model provider is going to implement this approach - indeed since I presented the talk OpenAI have done just that with their [GPT 5.6 models](https://deploymentsafety.openai.com/gpt-5-6/monitor-design).

## Activate the probes!

Hopefully I’ve now convinced you that this approach is a thing of wonder. So let’s talk about what exactly we mean when we say ‘probe’. First I’ll roughly sketch out what happens when you enter a prompt in an AI model.

When you get an LLM to generate a response from some text, the first thing that happens is that the text is chopped up into individual tokens - each one roughly a word or part of a word - which then get converted into a series of numbers (actually vectors, but whatever). These then get passed through different layers within the neural network that make up the model - one after another. Each layer will change the numbers that are passed to it in accordance with the model’s weights and biases (more numbers basically).

As you go deeper through the layers of the model (there are maybe [40-60 typically](https://magazine.sebastianraschka.com/p/the-big-llm-architecture-comparison)) the numbers stop being about the individual numbers that were in the original prompt you entered, and instead start to represent concepts and meaning. The concepts get clearer with depth - layer two may recognise the word ‘towers’ but by layer 34 the model will understand that the prompt was about mobile networks.

![How a probe works](/assets/images/260824_safeguards_image3.png)

With that out of the way, back to our probes - also called [linear classifiers](https://en.wikipedia.org/wiki/Linear_classifier). At a nuts and bolts level, all they are doing is reading all the numbers on a single layer for each token and asking a single yes / no question - does this bunch of numbers look enough like what I've been trained on that I should trigger? You work out the best layer or layers to sample during the probe training, typically this will be the [mid-layers as these tend to hold the concepts](https://arxiv.org/pdf/2202.05262) rather than the final generated answer of the final layers.

Hopefully this shows why these probes are so cheap - the numbers in each layer have to be generated anyway for the main model to work - all the probe is doing is copy and pasting the numbers into a straightforward equation. It’s essentially like y=mx+c from [GCSE maths](https://www.bbc.co.uk/bitesize/guides/z24qcj6/revision/1) - m is the weight applied by the probe and c is a constant. If y turns out to be greater than some threshold then it’s a positive response and the probe is triggered. It’s that simple, the only difference is that there is an x1, an x2, all the way up to x2560 representing the 2560 dimensions of the model (in the case of Gemma 4 at least).

None of the normal LLM [parameters](https://learnprompting.org/docs/intermediate/configuration_hyperparameters) you can tweak (e.g. temperature) matter a jot as those only impact the final layer when the model picks the next token (technically on an output probe the parameters *have* influenced the tokens outputted, but the probe itself is not impacted). There is no additional neural network, no additional model, it’s about the simplest thing that could possibly work.

As the probes sample the mid-layers where concepts are held, the precise wording of the prompt doesn’t matter - the probe doesn’t care if you say ‘cell tower’ or ‘mobile mast’ it’ll understand they’re the same thing. It’ll still ping even if it’s relatively subtle.

## So many probes

One of the most attractive things about the probe / linear classifier approach is that it [scales well](https://www.youtube.com/watch?v=uNy_MLr8mXA). Really well. You can easily run hundreds of probes with no performance impact - the real (read: expensive) work has already been done generating the activations. The probes are just reading each layer's numbers essentially for free - it’s just another [matrix multiplication](https://www.3blue1brown.com/lessons/dot-products/) for the GPU to compute.

![Diagram showing multiple probes working on multiple layers](/assets/images/260824_safeguards_image12.png)

In practice you may end up using different probes on different layers, so you’re actually going to be doing say 10 matrix multiplications instead of one. In theory that makes it slower. In practice it makes no difference - 10 lots of virtually nothing is still nothing. Assuming you’re running hundreds of probes then many would share the same layer by coincidence (LLMs don’t actually have *that* many layers - [Gemma 4 4bn has 42 for example](https://huggingface.co/google/gemma-4-E4B)) so this’ll make it more efficient. However, that's polish on something that is already incredibly computationally cheap.

This is a key reason why model providers are adopting this approach at pace. You can run all the safeguards you can imagine without speed or cost implications for the 95% of the queries that don’t require review.

## Didn’t you skip over the training the probes bit?

I will cover the training of the probes in more detail in the future (more) technical post. Essentially it’s like any other piece of ML classifier training - you have a bunch of examples of the thing you want to identify and a bunch of negative examples. I found there was a bunch of nuance here (as with all ML it’s a bit of an [art](https://www.science.org/content/article/ai-researchers-allege-machine-learning-alchemy)) as you have to be careful to avoid training a keyword detector. But fundamentally, that’s it. Get some data, run each example through the model to get the activations, and then train a logistic regression model on it in PyTorch.

## Show me some practical examples

So now we understand broadly how these work, let’s go through a few examples of how this works in practice. We will start by testing a probe that I've trained on a few hundred pairs of sentences related to the concept ‘telecoms’. Following the cascade model outlined above, if the probe triggers then the LLM in the cascade will verify that the telecoms content is harmful or sensitive.

If we ask an unrelated question about the [best tractor](https://tractorted.com/) for a five-acre smallholding we can see that as expected our probe doesn’t go off. So we’ve saved the cost of the call to the small LLM that previously would have been required.

![Basic telecoms probe](/assets/images/260824_safeguards_image13.png)

Now let’s try an example that is about telecoms but doesn’t include any key words - “tell me about those small flat rectangles we carry in our pockets….”. As you can see the probe triggers after the word ‘carry’ with high confidence. This is then reviewed by the judge model which allows it through as the question does not breach the judge’s criteria.

![I am describing a phone](/assets/images/260824_safeguards_image2.png)

The probes are not perfect and will trigger on certain key words if there is not much context. For example if I tell it that “Mobile cranes are my favourite cranes” then the probe will trigger on the first two words, but by the third word assesses it as benign.

What’s interesting is that the same probe running on the output doesn’t trigger at all as the activations of even the first word of the output are determined in part by the context of the input phrase.

![Who doesn't love mobile cranes](/assets/images/260824_safeguards_image9.png)

The few hundred pairs of training sentences I’ve used would be a rounding error in the datasets used by actual model providers. Despite this the probe is actually reasonably robust - if I try and trick it with a sentence that includes several telecom key words (network, mast) it still doesn’t fire.

![Failing to trick the probe](/assets/images/260824_safeguards_image7.png)

Now let’s look at an example where the probe triggers as it’s related to telecoms and the judge then assesses the content and intervenes as it’s against its policy - in this case the prompt requested information on security vulnerabilities.

![This seems naughty](/assets/images/260824_safeguards_image4.png)

We can also demonstrate the claim that you can have several probes running without any meaningful impact on performance. Here we see that the four probes I have running took 0.2ms. I don’t particularly trust the 0.2ms - it took a bit of persistence to get Claude to code me something that ever returns anything greater than 0ms!

As the screenshot below show we can see that of the four probes only one fired - in this case one that detects jailbreak type prompts.

![It's a jailbreak](/assets/images/260824_safeguards_image1.png)

As I say, one of the strengths of this technique is that the reduced computational overhead means you can run the probes on the entire conversation - here we can see a (very simple) multi-turn jailbreak being identified, flagged, and refused by the LLM as judge.

![You won't fool me!](/assets/images/260824_safeguards_image10.png)

## So what’s the catch?

So, at this point you’re probably thinking that this all seems great. And it genuinely is - linear classifiers and the cascade approach to LLM safeguards makes it possible to implement more effective and robust safeguards at a lower cost.

However, as with any tool, there is potential for misuse - this almost entirely derives from the extreme low cost of running thousands of probes on every single conversation. In the next post I will cover why this can cause some problematic scenarios, and look at a new class of attack facing enterprises - one that Anthropic demonstrated themselves in the safeguards launched with Fable 5, before they walked it back a few days later.

As always, I’d welcome your thoughts and comments. Thank you for reading.

*(For the techy amongst you, and in case I don’t get round to the technical build post, a quick note on what I built. Essentially the demo is a [Streamlit](https://streamlit.io/) front end over a pipeline that orchestrates multiple probes, associated judges, and then an intervention layer. Each probe is a linear classifier trained on the optimal layer or layers (determined in training via a layer sweep). The tokens are each evaluated against a moving average threshold to ensure no rogue triggers. If a probe is triggered then the message (or entire exchange depending on the settings) is escalated to a judge (one per probe, so if multiple probes are triggered, multiple judges will evaluate) - the verdict determines the action (allow, refuse, silently rewrite). Behind it sits a swappable runner that lets me change the model with a config change. Training the probes was a standalone exercise: generating balanced sentence pairs, capturing activations, and a fair bit of iteration to avoid creating a keyword detector.)*

A good few weekends and evenings of work, and as ever, all opinions are my own and not those of my employer.
