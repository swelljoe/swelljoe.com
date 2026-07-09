---
title: "The Elements of AI Style"
date: 2026-07-09T13:48:58-05:00
draft: false
---

![The Elements of Style cover with "AI" sloppily edited in](/img/elements-of-ai-style.jpg)

I don't like the verbosity of AI models and I don't generally like their prose. Some are better than others, some have better prose that makes it less annoying, some are less sychophantic, also less annoying. But, a universal trait of LLMs is they do go on. LLMs love to ramble and there's very little we can do about it.

But, I still try. My AGENTS.md contains the following, lifted directly from Strunk & White's Elements of Style:

> Omit needless words. Vigorous writing is concise. A sentence should contain no unnecessary words, a paragraph no unnecessary sentences, for the same reason that a drawing should have no unnecessary lines and a machine no unnecessary parts. 

Others use [caveman](https://github.com/JuliusBrussee/caveman) for the same reason, but that repo is strong evidence that it doesn't work. Also, someone needs to tell them that you don't need dozens of files to tell models to be concise.

I don't know if instructing for brevity works to make models smarter, as has been claimed. I don't know if it makes the models use less tokens, as has been claimed. Both seem plausible. And, there's [literature on the topic](https://arxiv.org/html/2401.05618) if you want to think deep about it.

But, I just want the model to not talk to me like I'm an idiot.

You could probably also just use, "Be concise." My hope is that quoting Strunk will inspire a spark of genius in the model and it will write in a way that does not grate, if it makes it faster, smarter, and cheaper, all the better.

The reason this is on my mind today, is that the new GPT 5.6 developer docs say to [avoid generic brevity instructions](https://developers.openai.com/api/docs/guides/latest-model#avoid-generic-brevity-instructions), which means that when I use GPT (rarely, mostly for Nelson because Opus refuses to do security work), I'll have to use a different set of instructions. Their recommended alternative seems to ignore what's wrong with models and why someone would want a generic brevity instruction.

> Lead with the conclusion. Include the evidence needed to support it, any material caveat, and the next action. Omit secondary detail and repetition.

This assumes the model has a bunch of really important things to say, and I'm just impatient and putting the point up front will solve the problem. GPT is historically the most egregious offender in terms of sychophancy, and middle of the pack on verbosity. Instructing for brevity tended to reduce both, so here's hoping one of the improvements in the new models is a lower tendency to be a babbling suck-up.
