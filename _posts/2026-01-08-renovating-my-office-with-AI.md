---
layout: post
title: "Renovating my office with AI"
---

My Christmas DIY project this year was to renovate my office. The biggest part of this was to do some Ikea hacking and add some built in cupboards, bookshelves, and a second desk. It struck me that Nano Banana Pro would definitely know about Ikea products and therefore might be able to help me visualise the end result (and importantly get approval from my wife!). 

My standard approach to DIY, is to come up with an idea, and then procrastinate about the details for days or weeks. I was true to form for this project, but eventually settled on a plan sufficient to knock up a rough scale plan in Visio:

![A terrible scale plan](/assets/images/0801_plan.png)

I then gave this to the model with a bit of additional detail:

`Here are my plans - I’m doing some ikea hacking. This is a front elevation of a set of built in cupboards and book cases i'm building in my office on the wall - the wall is 3.3m x 2.4m. As per the diagram there are a pair of white metod cupboards on each side, then a desk area in the middle. The outer cupboards are shaker style cupboards (2 doors) the inner cupboard will be 3 draws again in shaker style. The wall itself is covered in vertical panelling in mid grey. On top of the cupboards (the brown bit) are three pieces of laminate work top (oak). Then on top of the worktop is a  3 wide x 4 high kallax unit which has an open back so you can see the grey panelling. Nothing in the middle as it's a desk. Try and visualise it for me based on the picture.`

This came back with a good start (I also tried it without the image to see how much it was understanding from the diagram and the result was much worse) as below:

![The first attempt](/assets/images/0801_image1.png)

There was then a bit of back and forth to correct what it got wrong. Specifically, it hadn’t picked up (from either the text on the plan, or the prompt itself) that the Kallax were 3x4 units* and secondly that the desk would be lower than the tops of the cupboards. This points to the model very much still operating from ‘vibes’ rather than specific details, but a couple of nudges got it back on track and resulted in the picture below:

![The second attempt](/assets/images/0801_image2.png)

Which was close enough to submit for (and achieve) wife approval. Whilst the final result isn’t identical (I made some changes!) and I’ve not added door handles yet (brass? chrome? leave as bits of masking tape for 3 months whilst I ponder it?), I think it’s pretty close!

![The real thing](/assets/images/0801_real.png)

Hopefully, this post is underlining how far multimodal models have progressed in the past year. Submitting a plan and getting a half decent result would just not have been possible even a couple of months ago – the success of this points to Google’s success in integrating Gemini 3 Pro’s world understanding into the model. Clearly this isn’t good enough to replace an expensive graphical artist working for an interior designer, but that was never on the table for this project and it’s not *that* far off.    

As demonstrated in my [previous post](https://ewanpanter.github.io/2025-11-25/experiments-with-nano-banana-pro) about using Nano Banana Pro to visualise mermaid diagrams we are seeing image generation models move from fun toys to useful tools. Looking forward to whatever we have access to for my Xmas 2026 project and in the meantime please share any visualisations of your own projects! 



\* 3x4 Units that were 10cm too tall for the space – I can confirm that with the application of a table saw and wooden inserts you can make Kallax fit any space!

