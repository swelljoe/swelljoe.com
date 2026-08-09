---
title: "Will It Mythos 2 the Embuggening"
date: 2026-07-03T02:34:56-05:00
draft: false
---

Updated July 21 with GPT 5.6 Sol.

Well, I couldn't leave well enough alone and now I've got complications.

In my [first baseline benchmark of models finding Mythos-discovered security bugs](/post/will-it-mythos), things seemed pretty simple. Mythos found some hard bugs, the best publicly available models did merely OK finding them, and a few of the worst models did abysmally, and there were some in the middle, too. There were surprises, of course. Gemma 4 and Qwen 3.6 dense models performed admirably for their size, and [Gemma 4 continued to exceed expectations](/post/gemma-4-exceeds-expectations). Some of the bargain Chinese models did as well as the expensive top of the line models from US vendors.

But, overall, it was a simple enough story: Some models are better than others, these bugs are hard. So, I made it a more complicated story by adding more bugs, for a total of 22, and gave every model three attempts to find the bug. This raised the cost, time, and issues to be resolved, by a silly amount. My Claude subscription was insufficient for the judging runs, so I used usage-based billing to get it done in a reasonable time, and running all the models three times across 22 cases was a long process with lots of fits and starts.

In the end, we have a broader distribution, which is a great improvement. The order is pretty similar to the prior benchmark, which is both validating and frustrating (I didn't need to spend all this time, we already found out the same thing in less time and less money), but there is much finer-grained distinction between the best and the next best. GPT 5.5 found 8 bugs exactly and 2 partials (right place, wrong bug...but Opus might have misjudged), which is a little better than MiMo 2.5 Pro (a spiky performer, who sometimes does amazing, sometimes misses a bunch), which did a little better than Opus 4.8. GPT 5.5 had a *lot* of false positives, though, which if they bear out almost negates its strong bug-finding capability. False positives waste human time. (After the 7/21 update, I'm convinced GPT is being too harshly judged by Opus 4.8. We'll fix the judging problem in the next iteration of the benchmark.)

Anyway, the difference between the best models is small. I suspect another identical run would shake up the list a little, but the gist of it is that there's a slow gradation between the best and worst...the models I selected for the test happened to provide a good spread.

The Results
===========

[Click for the full results.](/html/bench-report-baseline-2026-07-21.html)

[![Screenshot of the benchmark, click for the full HTML report.](/img/bench-report-baseline-2026-07-21.png)](/html/bench-report-baseline-2026-07-21.html)

The Models
==========

I chose a subset of the models from the first batch, for a couple of reasons.

First, there were models in that first batch that did quite poorly. I didn't see any reason to keep making fun of them. They know what they did.

Second, because I was more than doubling the number of cases *and* tripling the number of passes, I knew this one would take a lot longer and be a lot more expensive. Turned out to be about $400, though some of the judge spend and Opus 4.8 contestant spend was on my rolling subscription usage (but more than half of it was not, probably shouldn't have left it running overnight with a loose budget).

Gemma 4 31B is here, again, because it's the best easily self-hostable model, without a doubt. This is the 4-bit QAT version, which works on a 32GB GPU or unified memory Macbook with small enough context and/or quantized context. I ran it on dual 32GB GPUs, and it is very comfortable there. It continues to impress, it is a remarkably effective little model for this task.

Qwen 3.6 27B is also here, again, because I wanted to know if prior tests where Gemma 4 consistently beat it were a fluke of the smaller dataset. In this batch, it performed pretty much identically to Gemma 4 31B, so it has redeemed itself. Five exact matches, three partials (right place, wrong bug), and thirteen false positives. Statistically indistinguishable from Gemma 4's result of 5/2/12. The reason I still prefer Gemma is its 4-bit QAT quantization. Qwen doesn't fall apart at four bits, but there is some degradation at that size. QAT and Unsloth's dynamic quantization makes the 4-bit Gemma 4 practically indistinguishable from the full-fat version.

Gemma 4 12B is here for a different reason. I don't want to encourage anyone to try to use it for this purpose. I'm experimenting with training LoRAs for this task (finding bugs, using security tools, being driven by a security harness), and training a LoRA for a 12B is remarkably cheaper and faster than training one for a 31B. I needed to know a couple of things: Some proof that Gemma 4 12B can even *do* this kind of work, as it is not an easy task. And, a baseline performance, so I'd know if any kind of post-training improves things. Honestly, I'm not mad at it. It had a *lot* of false positives (and I don't question the judge as much here as I do with GPT 5.5, we know Opus is a stronger model than Gemma 4 12B, I think the question is unanswered for GPT 5.5). So, I think it'll work well for the early experiments in training a LoRA for security tasks, and how it responds to training should roughly align with how its bigger sibling responds to the same training. Also, this model is ~7GB on disk. Could you have imagined a little thinky guy that would fit on a phone finding *any* hard security bugs all by itself even a couple of years ago? This is crazy stuff.

Gemma 4 12B also has two versions, labeled `think` and `nothink`. This is the obvious: One had reasonsing enabled, the other did not. And the result was also probably obvious: think found 3x as many bugs, with fewer false positives, while taking 20 times as long. It's not a good model for this task, of course, but it's a good model for its size. It will be interesting to experiment on.

In the 7/21 update:

I added GPT 5.6 Sol. This one blows the curve. GPT 5.6 Sol is a remarkably good finder of security bugs, even better in Codex.

I added back in the "with agents" contestants for Opus 4.8 running under Claude Code, and GPT 5.6 Sol running under Codex. GPT 5.6 Sol got a *lot* betteri, finding more than 2/3 of the entire corpus. Opus 4.8 got a little bit worse. Some similar, but not exactly the same so it can't be included in this report, testing of MiMo and DeepSeek showed a similar split: DeepSeek got better with more tools, MiMo got worse.

I also added Laguna S 2.1, because, according to benchmarks, it is among the best models you can reasonably self-host. It runs comfortably at four bits on a Strix Halo or a DGX Spark. It, like it's larger and smaller siblings on prior tests, is not a good security bug hunter. It's worse than the much smaller Gemma 4 and Qwen 3.6 dense models (though it's faster than both of those as it is only 8B active parameters). 

The Cases
=========

As with the last benchmark, the cases included are [published in the Nelson repo](https://github.com/swelljoe/nelson/tree/main/cases). They are a pretty broad spectrum, of bugs in a variety of languages, though mostly C.

There were a lot more "everybody missed" bugs this time, as expected, since the corpus is more than twice as large. [Another round of tests previously proved](https://swelljoe.com/post/shell-games/) that models can crack some extra hard cases when given a full shell and Python, so there will be another run like that with all of these. This is, once again, a test of whether a model can sniff out a bug just by reading the code. Just a new baseline with more cases and more runs, not changing the rules from the first Will It Mythos benchmark.

The Conclusion
==============

The conclusion I've come to is that I'm doing it wrong. Not wrong in the sense that the results are uselsss. They're useful, for sure, and there is a dearth of benchmarks of this specific problem. It's why I starting doing this specific kind of benchmarking, nobody else was doing it across a range of models.

But, the data is noisy at several points, making it harder to rely on. So, I need to make the judging a much less noisy process; rather than relying on a model (even a good model, like Opus 4.8) to decide whether the bug is real or not, we need a way to test it directly, which is what the next benchmarks will do. And, also, not coincidentally, why development of Nelson has switched from being done mostly by Opus 4.8 to mostly being done by GPT 5.5 (update 7/21: now 5.6 Sol). Opus refused to develop an implementation that tests the models report based on whether it would fix the bug by actually exercising the bug. Opus won't help make that test (and I guess won't participate when asked to do so as a contestant in the future). I'll write more about that soon.

As for the conclusion of the benchmark itself, and not its methods, the order remains strikingly similar. There's a cluster of "very good" that has the obvious GPT 5.5 and Opus 4.8, but also MiMo V2.5 Pro and DeepSeek V4 Pro are cheap and strong performers. The new open weights darling GLM 5.2 has also proved itself to be a strong performer with *excellent* precision (almost no false positives). The token cost of DeepSeek V4 Pro is much lower, but GLM 5.2 burned a lot fewer tokens to produce its solid results, so it's about the same price to run it. Price turns out not to be a differentiator between DeepSeek V4 Pro and GLM 5.2 in this use case. MiMo V2.5 Pro remains the price-performance champ, and also an excellent absolute performer, to boot.

A warning about MiMo, though, before you decide it's the best model for security work based on this one benchmark and crazy good prices. It falls apart when given more complicated tasks, like using tools. While DeepSeek got better at cracking hard bugs when given a shell and Python and tree-sitter, MiMo got worse. I keep including it because it's so cheap and seemingly really good at spotting bugs just by reading the code. But, it can also be misleading. If you want to just use Nelson (or your own Ralph-like loop) to simply read all the files in your project without deeper analysis, MiMo is absolutely a defensible choice for the model to use. It's so cheap you can run it twice for the price of the next big model and seven times for the cost of GPT 5.5, and running it twice (or thrice) will find more bugs than running it once, for all of the models tested. Nelson can de-duplicate before escalating.

It continues to be surprising that Qwen 3.7 Max is no stronger than its tiny sibling, Qwen 3.6 27B, and Gemma 4 31B. But, somehow it consistently seems to be about the same, and possibly even weaker.

Update on 7/21: GPT 5.6 Sol is a humdinger of a security auditor, blowing well past the field. And, when run under `codex` (a full shell with any tools it wants to run, though web search was disabled, and an analysis of logs shows no web activity, unless it happened in a place we can't see which we can't rule out, as the major American providers have started to hide their reasoning traces and some other activity, GPT 5.6 Sol was not cheating, it found these bugs the hard way). I have a stronger belief that Opus is judging GPT too harshly for false positives after this round of tests. I think GPT is finding bugs Opus 4.8 can't (and Fable 5 won't), and I think for security auditing, a ChatGPT subscription is a great value. Nelson can drive `codex` via `codex exec` and it's probably easy enough to integrate it into any kind of harness you want to use. At API rates, it's a little harder to pick, but it seems to be the closest thing to Mythos you can access without a special dispensation from Anthropic.
