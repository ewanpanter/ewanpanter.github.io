---
layout: post
title: "Experiments with Nano Banana Pro and Mermaid"
---

Since it was released a few days ago, I've seen a lot of examples demonstrating that Google's new Nano Banana Pro image generation model is both fairly decent at outputting text in images and sticking rigidly to the request.

With all image generation models the images you get out of the model are going to be dependent on you writing a good prompt - which can be tricky. An obvious use case I thought of was doing pretty slides of process flows, sequence diagrams and architecture diagrams (as an ex-management consultant I do love a slide!).

However it's pretty hard to describe a sequence diagram in natural language right? Well what if we don't need to? I currently tend to do this type of diagram in [Mermaid](https://mermaid.js.org/syntax/examples.html), which whilst not the prettiest\*, it is very good at describing relationships. Additionally, LLMs are also very good at Mermaid, so one of my favourite tricks to understand code is to dump it into Gemini and ask it to create me a Mermaid diagram of the process flow.

This then leads to the obvious question since Nano Banana Pro is still a LLM, and understands Mermaid, is it possible to dump a bunch of Mermaid into the model and have it spit out a pretty version?

So - here is an example mermaid sequence diagram:

![A example sequence diagram in Mermaid](/assets/images/sequence_diagram_mermaid.jpg)

And here is the model's attempt (I told it to use Google corporate colours - it is very good at following style guidelines):

![A LLM version](/assets/images/sequence_diagram.jpg)

It's actually pretty good! I might change where it's written 'Notifications' and 'Response' but perhaps that's being picky.

We can also try other types of diagram. Here's the Mermaid version:
<!--more-->
![A example sequence diagram in Mermaid](/assets/images/state_diagram_mermaid.jpg)

And here's the Nano Banana Pro effort (this time in .gov styling) - again not bad. 

![A LLM version](/assets/images/state_diagram.jpg)

I figured we might as well go a step further, and see what happened if I just dumped a load of code in and told it to make a flow chart explaining it. I used a few files that added up to about ten thousand tokens or so - so small by code base standards. Unfortunately this caused the model to crash repeatedly after thinking for 5 minutes - so maybe don't do that! Stick with getting a normal LLM to generate the mermaid and then getting Nano Banana to make the pretty picture.

All in all not bad! It's not massively cheap (this experiment cost me about a quid), but I expect the cost to come down and the quality to go up. Clearly the danger here is where the model is 98% accurate - that's accurate enough to assume it'll do a good job, chuck it into a presentation without checking, and then someone spots the mistake in the meeting and makes you look silly. Buyer beware I guess!

\* I know you can format Mermaid diagrams to make them look pretty, but a) it's a pain b) the arrows still go over things.