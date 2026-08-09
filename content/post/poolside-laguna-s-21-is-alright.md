---
title: "Poolside Laguna S 2.1 Is Alright"
date: 2026-07-23T22:37:20-05:00
draft: false
---

I try every new small model that comes along. At the very least, I'll give it a run through my Nelson security auditing benchmark suite to see if the new thing can topple the current tiny model champs, Gemma 4 31B and Qwen 3.6 27B, which are very closely matched and punch well above their weight, defeating several much larger models. Poolside has released a few models, and I've tried Laguna XS.2 (a 33B-A3B MoE, easily self-hosted on modest hardware) and Laguna M.1, both of which did quite poorly on the security auditing benchmarks, and didn't have much to recommend them over the previously mentioned Gemma 4 and Qwen 3.6 models, so I never did anything else with them.

Laguna S 2.1 arrived a few days ago, though, and the benchmarks looked amazingly promising. The first new self-hostable model that ranks higher than Qwen 3.6 27B on several coding benchmarks, so it looked very promising. So, I tried running it locally, and failed a few times. `llama.cpp` needed patches, the GGUFs were botched, etc. but I finally  got it sorted on the second day and could chat with it in the web UI. But, when I tried to use it in `pool`, the Poolside CLI agent, it got lost in tool calling loops and other weird situations, so for the benchmark, I used the paid model on OpenRouter (there's a free option right now, as well, if you want to try it yourself).

It did quite poorly on the Nelson benchmark, finding only 2/22 bugs vs. Gemma 4, which found 5/22 plus 2 partials (right place, wrong bug), and Qwen 3.6 27B with 5/22 plus 3 partials. Even the tiny little Gemma 4 12B model found 4/22 and 3 partials (but a ton of false positives, so it's not really better). Laguna S 2.1 is much bigger, but performed worse.

So, I'm not immediately on board with this new model. I want to like it, but for one of my primary use cases, a thing I'm most interested in for small self-hosted models, it's just not very good.

But, it can probably do other stuff, right?

So, today, after finally getting it up and running properly on the Strix Halo in the 4-bit quantization (unsloth GGUF in current llama.cpp, forced `--yarn-attn-factor 1.0` as the GGUF ships with the wrong value, which causes the weird tool loops and other insanity, I started using it. I set it the task of fixing a bug in [Prose or Con](https://prose-or-con.com/), a game I made to test whether people, including me, can detect AI writing just by reading it (it seems like people can, to some degree, you should give it a try). The bug is simple, though fixing it in the most satisfying way is pretty challenging. On the Sources page, we currently list every human source in the corpus with a link to its origin; in some cases, this is required by the license of the text, but mostly it just seemed like the polite thing to do. But, for the AI passages, the page just listed the name of each model, e.g. `Claude Opus 4.8 — Claude Opus 4.8 anthropic/claude-opus-4.8, AI-generated`, over and over for every item in the corpus produced by that model. Looks a mess, doesn't provide any useful information. I could have just made it prune the list of models down to a list of each model that was used, but I wanted something more challenging, and it seemed like a good idea to make it behave the same as the human sources. So, I asked Laguna to make each model entry a link to the passage.

It struggled. It took a good long while. It also maide some interesting choices. Though I would assume it has some kind of edit/patch tool built into `pool` (I haven't checked, I don't know much about `pool`), it used Python to patch the TypeScript files. It wrote a Python program every time it wanted to edit a file. But, it did the job! First try, no less. It caught itself when things went wrong and recovered. It has a dogged determination that I've come to expect from Opus but few other models. It was going to finish the job I gave it, hell or high water, and it did. And, it did a good job. My description of the task was brief, I didn't tell it how to do it, I just told it what I wanted it to look like, and it did exactly what I wanted. No back and forth. No requests for corrections. It just did the job.

Here's the summary it gave me at the end:

```
• Here's a summary of all the changes I made to fix the bug:

  Problem

  On the "Sources" page, AI-written passages had no identifying information — they all just showed "Claude" as the title, because meta.title was absent for AI-generated passages and
  sources.json had no text to derive a title from. The entries also weren't clickable to view the passage text.

  Changes

  1. corpus/build/build-corpus.ts — Derive AI passage titles at build time

   - Added a deriveTitle() helper that takes the first 8 words of the passage text and appends an ellipsis if truncated.
   - Modified sourceOf() so that AI passages (which have no meta.title) get a title derived from their full text. Human passages keep their real titles from metadata.

  2. web/public/sources.json — Regenerated

   - Ran a regeneration script against the existing corpus.json (the committed monolith) to produce a new sources.json with title populated for all 588 AI entries. The JSON shape is clean (no
  null fields for missing optional values).

  3. web/src/views.ts — Link AI titles + new renderPassage view

   - renderSources: AI entry titles now link to #/passage/<id> (using encodeURIComponent for the id). Human entries keep their existing behavior (external URL link or plain text).
   - renderPassage (new exported function): Fetches the passage via the existing getItem function and renders it the same way the game's reveal screen does — passage text as paragraphs,
  attribution line (source, title, author, model, license, "view original" link) — but without any "Human / AI" choice buttons or verdict. Includes a "← Back to sources" link.

  4. web/src/main.ts — New route

   - Added import { renderPassage } and a #/passage/<id> route handler that decodes the id, looks up the matching SourceEntry from the cached sources list, and calls renderPassage with the
  existing getItem (corpus item fetcher and cache, already wired up).

  All TypeScript type-checks pass (.tsc --noEmit clean for both the web and corpus projects), and the Vite build succeeds.

• Worked for 60m 46s
```

A SoTA model would have done this in 10-15 minutes. I could have done it myself in an hour. But, I can't complain. It's a good little thinky guy in the computer, it operates almost, but not quite, fast enough for comfortable interactive agentic use at 20-25 tokens per second, as long as you don't mind flipping to another tab to work on other stuff while it chews (but I do that with Claude and Codex, too, even though they're much faster).

Anyway, this isn't a benchmark, I was just checking the vibes. Another project I gave Laguna S 2.1 (though in the OpenRouter-hosted variant) was to [implement the `pool` resume backend for `flar`](https://swelljoe.com/post/i-let-every-agent-implement-its-own-flar-backend/). It did OK, though it did generate more code review comments from Copilot than most of the other models did. It got a bonus point for adding a new test file for `main.go` that no other model thought to add.

Determined, sure, but, is it smart?
===================================

I also spent an hour or so chatting with it about stuff. I asked it some of the same kinds of questions I ask other models, when I want to test their smarts and how much they hallucinate on technical topics. I ask about how models are trained, what quantization-aware training means, how LoRAs are trained, and I'll ask whatever other questions I've recently posed to the big models. Once again, not a benchmark, just vibes, but I think Gemma 4, at least in the biggest dense version, is "smarter". It seems to know more. But, here's where determination has value. Laguna doesn't let lack of knowledge stop it from answering. It searches the web, unprompted (I have Exa MCP setup in llama.cpp), and figures something out. It seems to think through things well.

If I absolutely needed a self-hosted model for security or compliance reasons or something, maybe this would be the one? I'm still on the fence about whether it's as capable as Gemma 4 31B, but it's a little bit faster on the Strix Halo, despite being more than three times as large, as it's an MoE with only 8B active. The low performance on the Nelson benchmark and the weird way it uses Python for everything gives me pause.

But is it better than Gemma 4?
==============================

Writing the previous paragraph made me curious enough to find out if it's really better than the biggest Gemma 4, which often surprises me by how well it reasons about stuff, how well it uses tools, etc. So, I gave Gemma 4 (with Zed, as the agent, because that's what I happened to have preconfigured for Gemma on localhost) the same exact prompt. It "finished" much faster. Like ten minutes and half the tokens. But, it was incomplete and untested. The implementation was mostly there, but I needed to ask it to regenerate the `sources.json` from the `corpus.json` to make links work (which required a script that Laguna just made on its own), and there were two bugs that took a few back-and-forth prompts to fix (mostly just "I got the same bug" or "here's the new bug"). Laguna acted like a big model: It did everything itself. It wasn't *good* at it, it flailed around a lot more than I'm used to seeing Claude or GPT models do, and more than Gemma 4 did. But, it did the job with no extra prompting. It shipped a working feature that I could probably push to production.

And, in Copilot review, Laguna's PR produced 2 comments, both minor cosmetic bugs. Gemma 4 produced five comments, one allegedly security (but not really, since we control the inputs in the current implementation). Gemma 4 also introduced a real bug in the `build-corpus.ts` that would make it fail to run in the usual case. That's more serious and would be an annoyance at some point in the future.

I think I have to conclude Laguna S 2.1 is a better coder than Gemma 4 31B, even though it struggles with basic stuff.

As an aside, I also let Laguna try it in the Zed agent, out of a sense of fairness. Maybe it'd find/use more tools there. But, it seemed to have an even harder time there, still doing everything with Python scripts rather than the tools provided by the agent, failing at that repeatedly and finally trying the edit tool and repeatedly failing at that, as welli, seemingly a problem with the format of tool output in Zed Agent (it kept complaining about `<thead>` being unexpected, etc.). Laguna's biggest weakness is seemingly tools. I love how stubbornly it completes the task, despite all that, but I feel sorry for how hard it worked only to fail in the end.

