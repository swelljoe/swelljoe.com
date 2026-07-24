---
title: "I Tried (Almost) Every Major Model Subscription So You Don't Have To"
date: 2026-07-23T18:16:48-05:00
draft: false
---

The best way to buy AI inference is usually a subscription. It's well-known that a $20 or $100 or $200 subscription from Anthropic or OpenAI is the most cost-effective way to buy access to their models, by far. One report said a $200 subscription could provide ~$9000 or ~$12000 worth of tokens at API rates from Anthropic and OpenAI, respectively. I haven't done the precise math on this, but I have tried nearly every AI coding subscription now, as I subscribed to almost everything to run new benchmarks of security vulnerability auditing capabilities of the major models using their own preferred agent (the one made by the model vendor, like Claude Code, or one built for and seemingly having some kind of nod as being a preferred option, like Reasonix) as harness.

The vendors don't really want you to be able to compare plans, so they use all sorts of obfuscations, and there have been accusations of moving the goal posts based on load or other factors. But, when you use them for a while, you get a feel for how much you're getting for your money, and people talk.

So, I've got the vibes of them. Some are much more generous than others and I'm here to tell you about it.

The Contenders
==============

For my benchmarks and testing, I subscribed to the following plans:

 - Claude Max ($100)
 - ChatGPT Plus ($20)
 - Google AI Pro ($20)
 - Kimi Moderato ($19)
 - Qwen Standard ($18)
 - MiMo Lite Token Plan ($63 for one year, but it's just a big pile of credits, not a rolling subscription like the others)
 - Github Copilot Pro ($10; I get this one free because of my Open Source contributions)

The Short Answer
================

The best deal right now is the ChatGPT $20 plan (or the bigger plans, if you use enough to warrant it). It gets a generous portion of GPT 5.6 Sol, a SoTA model, competitive with the best in the industry; maybe slightly worse than Fable for some tasks, but better for others, and not nearly as hobbled by guardrails that causes Fable to bow out of tasks mid-conversation at an annoying rate. I don't think there's any reason to pick anything else if you've only got $20 and want to use very good models.

If you want a model by a non-American company, MiMo, DeepSeek, and Qwen are good and a good value (but see below for caveats). GLM 5.2 is also a very good model if you don't need vision, but I've read their coding plans aren't a great deal, so I've skipped trying it.

Anthropic is probably still the best coding experience, but if you can't afford the $100 plan, OpenAI has a better offer for you.

The Long Answer
===============

Everybody's talking about Kimi K3, the Fable killer. Well...

![Image of Kimi subscription usage, showing current usage limit at 100% and weekly limit at 72%](/img/kimi-usage-joke.png)

Kimi K3 is unreasonably stingy in their coding plan and ends up being unreasonably expensive at API rates, when you inevitably run out of plan usage. In my testing, one tiny task chewed up the whole five-hour limit and a third of the weekly limit. Kimi's coding plan is the worst deal going right now, much worse than Anthropic's pricey plans. For the benchmark run that barely touched any of the other plans I tested (except Anthropic), I interrupted it after $100 of "extra usage" when I realized it would cost over $500 for the K3 run alone (Claude Opus 4.8 cost one five-hour limit and an extra ~$50). Kimi is also overwhelmed currently, so K3 is running very slow, and there are several hours out of the day where you get frequent 429 errors. I don't have a good feel for whether K3 is as good as the benchmarks promise. My gut is telling me "no". I've been told the Kimi plans in China are a better deal, but there's literally no reason for a western user to choose Kimi K3 at this time. It's just a really bad deal. I'll be canceling this plan, and I'm not recommending it.

If you want a model as strong as Fable and you're in the US, just buy from Anthropic. Their $100 and $200 plans are a fair deal. Not as generous as the OpenAI plans, and the $20 plan doesn't get you Fable while OpenAI gives you a generous amount of Sol on their $20 plan. But, I believe Anthropic is still the stronger coding ecosystem. The gap is smaller than ever and GPT is faster, though.

Github Copilot Pro used to be the best deal in the game, bar none. Extremely generous usage of every top model, plus unlimited usage of a handful of small models, like Raptor Mini (a GPT-derived model). It's since been nerfed to the point of being hard to recommend. Usage is stingy and the best models are a couple generations behind the frontier (e.g. Opus 4.6 is the best Anthropic model in their lineup). I still use it because I get it free due to my Open Source contributions over the years. But, I can't recommend folks pay for it as your primary AI subscription. I wouldn't call it a rip off, as Copilot PR reviews are genuinely really well-done, but $10 doesn't go far for agentic use.

The MiMo token plans have weird accounting (they give you "credits", they don't renew daily or weekly, it's just a bulk purchases of "credits") and isn't as generous as their low-cost token rates would make one assume. But, they're also not expensive. A $63/year token plan gives you a lot of usage, the bigger plans give you a lot more usage. A fine deal on a fine model. MiMo is slow, among the slowest of the major models. I'll only re-up this plan if I find I'm still interested in benchmarking Xiaomi models. I don't find it strong enough for daily work and the slowness is annoying. It is among the best deals for security auditing, though. Very strong vulnerability detection with minimal false positives at a very low price, if you have time to wait.

Google's $20 AI plan is just OK. Their models are currently insufficient for coding. They have good vision and sound, good generative image, video, and audio if that's your bag, and Gemini 3.6 Flash is fast. And, following Google tradition, they killed a good product (Gemini CLI) and replaced it with a much worse one. Antigravity is among the worst of the agents from major providers, intentionally hobbled for security work. I simply can't recommend Google AI subscription for a developer, because they don't have a good coding model, their agent stinks, and they're just not very serious about address either issue.

Qwen is the buggiest user experience of all of them at every stage. Their login CAPTCHA doesn't work with Firefox, their CLI agent is buggy as hell and fails to authenticate most of the time. The UI is confusingly labeled, and because authentication is buggy, you have to find your way through a dozen different kinds of accounts and endpoints to find the one that's actually right (and you might find the right one and it might be bugged in such a way that it fails anyway, so you cross it off the list of possibilities...only to find it was just the shitty agent being shit). Qwen 3.8 Max (preview) is a pretty good model. I'd been writing that it was a very good model, but then I looked back at some code it wrote and found that it didn't even test it and was broken. So...it's a sloppy model with not enough ambition. You know how I said Antigravity is among the worst of the agents? Qwen Code is *the* worst of the agents.

DeepSeek is Always an Option
============================

DeepSeek doesn't have a subscription, but their tokens are cheap at API rates, and with the Reasonix agent, you'll see extremely high cached tokens, which is even cheaper. You can run DeepSeek V4 Pro or Flash or a mix of the two, and run them hard, for something like a buck a day. If you don't use LLMs often, and your needs aren't extreme, this might be the way to go. If you need an API for custom usage, this is also maybe the way to go. It's a very strong model for the price. DeepSeek is fast. On par with Qwen, but Qwen 3.8 Pro Preview is a stronger model and burns fewer tokens on hard tasks. DeepSeek is expected to release a new iteration of their models soon, which may level the playing field, but even now, it's well-placed as a cheap and cheerful model you can use anywhere without worrying about it doing something stupid or costing a stupid amount of money.

But, What About xAI and Meta?
==============================

Be serious.
