---
title: "Lessons From CS2680"
date: 2026-11-25
authors: ["Denny Cao"]
summary: "Lessons from building and optimizing coding agents"
tags: ["agent-design", "testing", "optimization"]
toc: true
draft: false
---

Over the past four assignments, I've spent a lot of time trying to make coding agents more effective. Going into these assignments, I expected that most improvements would come from better prompting, expecting the frontier models to be able to do most of the heavy lifting. Instead, many of the biggest improvements came from fixing mistakes in the surrounding infrastructure, measuring the right things, understanding why certain optimizations weren't working.

<!--more-->

## 1. Arbitrary Limits

One mistake I kept making was introducing limits that seemed reasonable but ended up preventing the agent from solving tasks.

In Assignment 2, my harness had a token budget that counted the entire conversation history every time it was resent, so one task was terminated after supposedly consuming 618k tokens when its actual context was only 37.6k. In Assignment 3, Claude suggested limiting tasks to 40 model calls to save time. This configuration solved 0/26 tasks: successful tasks had needed 49–61 calls. In Assignment 4, aggressive early termination rules resulted in 0/26 solves in just two minutes.

A lot of times Claude would just suggest arbitrary limits, and I would fall into the trap of just following what Claude said. However, limits should be based on actual measurements. In Assignment 5, I examined completion times before choosing limits.

## 2. Test Feedback Is More Valuable Than Advice

Giving an agent better instructions was often less effective than giving it concrete feedback about whether its solution worked.

In Assignment 2, none of four instruction-based fixes for a repeatedly failing task worked. What helped was a check that failed when the agent tried to finish without satisfying the requirements. In Assignment 3, after I added a testing baseline, new test failures on one task went from 35 to 2 to 0. In Assignment 4, fresh retries graded by `evaluate()` converted 7 of 11 failed tasks into successes.

The most surprising result came in Assignment 5. Losing `evaluate()` was a big part of why my agent solved just 3/26 of the same tasks, compared with 7–12/26 in Assignment 4. Without feedback, we open the possibility of pursuing a bad trace, without knowing that anything is going wrong.

## 3. Benchmarking

In Assignment 4, I used SGLang's serving benchmark, configured to approximate my agent workload (25 conversations with 26 turns each). The results were dramatically different from the real agent. The benchmark didn't reproduce the way my agent resends previous messages, which lets SGLang reuse cached prefixes. A replay benchmark built from actual agent conversations gave me a more representative way to evaluate engine optimizations. This could have been due to Claude's own errors, but it really showed the importance of making benchmarks realistic to the tasks we are solving. 

## 4. More Concurrency $\neq$ More Throughput

For Assignment 4, one of my submissions had an admission rule to reserve 15k tokens per task, allowing all 26 tasks to run concurrently, but conversations grew to around 27k tokens each. The resulting cache evictions dropped hit rates as low as 3%, and requests took up to 67 seconds. Claude then suggested for the next version to reserve 30k tokens, limiting the agent to about 13 active tasks. Despite running fewer tasks at once, cache hit rates improved to 93–98%, and it solved 17 tasks in 60 minutes locally.

I had assumed that maximizing concurrency would maximize throughput, but once memory became the bottleneck, it made every request slower. I saw the same effect in Assignment 5, where running 24 tasks at once slowed model turns.

## Final Thoughts

My biggest takeaway is that building effective coding agents involves much more than making the model itself smarter. Many of my failures came from relatively simple systems decisions: counting tokens incorrectly, terminating tasks too early, running too many conversations concurrently, evaluating optimizations using unrepresentative benchmarks, etc.. If I approached these assignments again, I would establish reliable measurements before introducing optimizations.
