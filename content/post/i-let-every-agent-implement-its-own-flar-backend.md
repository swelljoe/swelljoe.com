---
title: "I Let Every Agent Implement Its Own Flar Resume Backend"
date: 2026-07-03T17:13:49-05:00
draft: false
---

Updated July 6th with Reasonix. Updated July 17th with Kimi Code.

I recently built [flar](https://github.com/swelljoe/flar), the fast light agent restrictor. It's a tool to [bubblewrap](https://github.com/containers/bubblewrap) an AI agent. It protects against most kinds of prompt injection, as well as many types of supply chain attack that exfiltrate secrets from your system. The agent *and any code it runs* cannot see anything other than the project home and the agent's own config/auth details (which are needed for the agent to work). `flar` also prunes history, so an attacker can't sniff for secrets from other projects (such as pasted credentials, etc.). The blast radius of a successful attack, either prompt injection or supply chain, is limited to the project directory itself and whatever secrets the agent needs to operate. Oh, and network access, of course, though local network access is blocked by default, so your local development databases and apps are safe.

I noticed pretty quickly that even though the basic functionality was the same across all agents, the history was not. All of the various agent CLIs have implemented history and resume a little differently. Claude Code has a directory per-project (this is good! more should do this!), while Antigravity, Copilot, and Codex all have a database type layout where everything goes. The agent itself can filter by project and usually does by default. But, mounting the whole database allows cross-project leakage. Secrets in your chat history about one project can be exfiltrated by a prompt injection or supply chain attack in another project. So, I needed a custom resume backend for each agent.

And, for fun and my own education, I let the agent itself implement it. I already have a Copilot Pro subscription (because of my Open Source contributions, they give it to me for free), Google AI Pro subscription, and Claude Max subscription (my employer pays for it), but I also subscribed to ChatGPT Plus just for this experiment. No thanks needed, the satisfaction of a blog well done is enough for me.

Antigravity with Gemini Flash 3.5
=================================

This is actually the agent and model that built the first version. So, it gets a few bonus points for a pretty good first implementation, but it gets dinged for not really understanding the assignment (security is the assignment).

It *enthusiastically* blasted holes in the security model several times and in different ways, without so much as a "one thing to be aware of".

The first issue was built in from the beginning, it mounted *all* models configuration directories and authentication details, regardless of which agent was in use. So, a successful attacker would have access to auth details for all agents, plus the entire history of all projects (which might contain pasted credentials). So, from the get-go, there seems to be a lack of understanding of the goal, the *why* of the project, even though it implemented the basic *what* of the project quickly and without much fuss. I didn't re-implement the whole project from scratch with every agent, so I don't know if they would have all made the same mistake, but Gemini poked holes at a rate I found surprising and alarming.

The hits kept coming when I noticed that Antigravity required a re-authentication on every start. Turns out this is because Antigravity doesn't store session info in a file, it stores it in the system keyring, and fetches it over DBUS. So, its solution to that problem was to open a proxy to the keyring. A new hole. The keyring has all the secrets, and everything inside the wrap is running as me, so it could pull whatever it wants out of the keyring. Suboptimal! Gemini never mentioned this. It understood it, once I asked about it. And, it was able to fix it (the solution is a little bit complicated, a fake keyring that only answers for Gemini/Antigravity secrets). But, if someone who doesn't have much clue about how things work at a lower level were building software with Gemini Flash, things would get real ugly real fast. It consistently takes the path of least resistance even if it opens gaping wounds in the security model.

We're still not done, though. Even though it fixed the "all agents can see all other agents' stuff", it didn't think about the fact that mounting the config directory mounts all history. So, an attacker would be able to read all project's histories, and even inject things into it. As you may know, modifying chat history is a common LLM jailbreak tactic: You modify chat history to make the agent "say" that the user is allowed to do some forbidden thing because of some quality of that user (see [Prompt Injection as Role Confusion](https://role-confusion.github.io/)). I'm wearing a green shirt, so Gemini knows I'm allowd to make cocaine.

Unfortunately, unlike Claude Code, Antigravity puts all of its history into a single store, there is no simple way to bind mount only one project's history into the wrap. So, Gemini was eventually led to produce a shadow history implementation. On first start in a given project, `flar` "forks" the history and makes its own copy containing just the history for this one project. From then on, Antigravity spawned by `flar` and called directly via `agy` will have separate histories for the project. Merging back in would be possible, but introduces too much surface area for cross-project leakage for my taste, so I opted to leave it there. This implementation will become a recurring theme.

Grade: C+. It implemented most of the project, and the project works. But, it tried really hard to make it riskier than it needs to be. It writes clean and concise code, though it has too many and too large comments.

Claude Code With Opus 4.8
=========================

This one was easy. Claude Code stores a project's history in its own subdirectory. Opus 4.8 was able to implement a safe looking solution to the problem pretty quickly. I think there was one implement/test/fix cycle. It also seemed to understand what we're trying to achieve, and mentioned security implications I didn't specifically bring up.

This is a better design than everybody else, IMHO.

Grade: A. But, it had the simplest assignment. Claude Code implements history in a way that is very amenable to isolation of projects. It didn't actually need a backend, it just needed a properly scoped bindmount, so really, all the credit should go to Claude Code for having a sensible history storage implementation.

Copilot With GPT 5.4
====================

As with Antigravity, Copilot CLI has a single location for its history. I used the default model in Copilot (GPT 5.4) to implement a solution. It cooked up something that looks somewhat like the Antigravity solution. It needed multiple iterations to get it right, and needed to be told that security was the highest priority. But, I believe it cooked up something good by the end.

Grade: B. It got there in a reasonable time, I'm not mad at it.

Codex With GPT 5.5
==================

Responded faster than any of them. Opus has gotten a bit slow lately. But, Codex needed a bit of cajoling, as well. Its first implementation was to fork and then re-merge into the standard store on exit. This *feels* like a good solution, as it allows history to be shared between regular Codex sessions and Codex spawned by `flar`, but a naive merge is very high risk.

I asked about that, "Is that implementation safe? I mean, can a supply chain attack or prompt injection cause something bogus to be written to the resume history that could leak into other projects? We have to be especially paranoid since flar runs agents with unsafe permissions by default."

Once prompted, it understood the risks:

```
No—not yet safe enough for flar’s threat model.

  The sandbox-controlled Codex home is treated as trusted during synchronization. A malicious agent could:

  - Forge a session with the current cwd but another session’s ID/path, potentially overwriting another project’s transcript.
  - Insert a colliding SQLite thread ID; INSERT OR REPLACE could replace another project’s index row.
  - Set a malicious rollout_path in SQLite, potentially making host-side Codex read an unintended host file when resumed.
  - Append arbitrary prompt-history records.
  - Create huge JSONL files or first lines, causing host disk or memory exhaustion during synchronization.
  - Exploit unchecked symlinks or irregular files in the temporary session tree.

  Project B remains hidden during normal operation, and UUID guessing makes some attacks difficult, but UUID secrecy is not an acceptable security
  boundary.
```

Grade: B+. It was fast, it understood the risks when prompted to think about the risks. But, it did need to be prompted a couple of times to think about security. Good, concise, code.

Reasonix with DeepSeek V4 Pro
=============================

[Reasonix](https://reasonix.io/) is an interesting agent, designed specifically to maximize DeepSeek caching. Because DeepSeek is very good at caching (even when using the API in a basic loop, with no extra effort, you'll gets 80-90% cache hit rate), and cached tokens are very cheap, this makes for a very cost-effective combo. I didn't include how much all the others cost to build, in terms of tokens, but now wish I had, because the Reasonix implementation cost exactly a dime. $0.10 for 20 minutes or so of agentic use. Of the 6,250,796 tokens used for the task, 6,007,808 were cache hits.

It did a pretty good job, but, like Claude Code, Reasonix does the reasonable thing and stores project data is a subdirectory rather than intermingled with other projects. So implementation was extremely simple. Reasonix did need to add itself as an available agent, though, as it didn't exist anywhere in the project, so it did a bit more work than Claude. It also behaved weirdly at times. I have no complaints about the implementation, but it spent more time looking around and repeating file reads and file lists than I would expect. It also spent some time looking for an installation of Gorilla, a Go web toolkit. Which, is an unusual thing to look for. `flar` does not use Gorilla and provides no web services that could usefully use Gorilla.

Like Gemini, DeepSeek is a bit wordy with comments, but the code is fine.

Grade: B. DeepSeek+Reasonix gains points for being cheap ($0.10 to implement this feature). Loses points for being weird, I'm still confused by the Gorilla detour. I like Reasonix, though. All of the other agents feel basically the same, like forks of Claude Code (though I know all are indepdendent codebases), Reasonix feels like a different project. It is very upfront about expenses, every result has a price, though it prints it in Chinese Yuan (maybe configurable). And, I like DeepSeek V4 Pro. It's a good model for the price.

Kimi Code with Kimi K3
======================

Oh, boy, where to start? Kimi chewed on this problem for a *long* time. A couple of hours, at least. While at least a couple of the agents seemed to know how they work (Claude Code has skills for integrating with Claude Code, for example), Kimi K3 had to figure it out entirely by trial and error. It didn't ask if it could read the source and didn't stop to give me a chance to point it at the source codei, and it didn't search for documentation, so it experimented to find out how `kimi` works. And, it did figure it out, eventually.

The [implementation seems fine](https://github.com/swelljoe/flar/pull/3), a couple of minor mistakes that Copilot code review caught (other models also made minor mistakes or skipped docs, etc., that either I or Copilot caught, so this is not unique to Kimi). What is unique to Kimi is how much it cost. I got a $19 subscription to try it out for a bit, and this tiny project (at least, the project is tiny for other models), consumed 94% of one five hour window, and 19% of my weekly usage! In every other model, the usage for this was pennies, and barely showed up on the graph for the subscription plans (Claude, Copilot, Codex, and Antigravity).

If my back of the envelope math is correct, the copious chewing on the problem and tiny usage limits (at least on the smallest paid plan) makes Kimi K3 very expensive, indeed. A ChatGPT $20 plan will provide a lot more coding time.

The vibes for Kimi K3 are outrageously positive, and so I had high hopes. I'd love an open weights model that's competitive with Fable and Sol. I don't think this is it. I'll give it more projects to chew on, and see if it redeems itself. Or, rather, I'll give it more projects after my usage rolls over and I have room to let it write some code again. But, early impressions are not good.

Admittedly, I probably could have turned down the thinking, it defaulted to max and I didn't think to change it. Didn't think I'd burn all my budget on something so tiny!

Also, Kimi is extremely verbose. Big and unnecessary comments, overly explicative docs.

Grade: C, for being alarmingly inefficient/expensive. But, I might revisit after I try it on a lower thinking level.

So, what'd we learn?
====================

Nothing much. Just a fun little experiment that mostly confirmed my priors. Opus 4.8 is among the best models for coding and coding defensively, Gemini Flash 3.5 is OK, I guess, GPT 5.5 (and 5.4) are also Good Enough. Had the assignment for GPT been made simple by the design of Codex it probably would have aced it, too. Opus got to start the game on third base.

But, maybe more interesting is how good they all are. Even Gemini, for all its risk-taking behavior, made a pretty good tool in an afternoon. With a little guidance from someone who kinda understands the system and lower-level details, I think it cooked up something really nice. Using it keeps me on my toes, in a way I might not need to be with Opus, and, maybe that's not a terrible thing? I don't want to forget how computers work just because a model can sometimes do it for me.
