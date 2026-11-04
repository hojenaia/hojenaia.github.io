---
title: "Qwen3.8 Flash-Next on an RTX PRO 6000: the missing tests"
date: 2026-11-04T12:00:00-03:00
description: "I tested Qwen3.8 Flash-Next Q2_0, IQ3_S, and UD-Q4_K_XL with Basalt on an RTX PRO 6000 Blackwell: speed, VRAM use, and coding results."
summary: "On a 96 GB RTX PRO 6000, I compared three Qwen3.8 Flash-Next quantizations with Basalt. Q2_0 was fastest and all passed the hard canary; Q4 fit comfortably but took longer on the coding tests."
tags: [AI, benchmarks, hardware]
draft: false
---

## Before the numbers: local models, Qwen, and Basalt

Running AI locally has become much more interesting, but it still means choosing models, quantizations, inference engines, and hardware. A cloud service hides most of that: you send a prompt and get an answer. At home, you can keep data under your control, use a model without relying on an external API, and experiment without paying per token. In exchange, you need enough memory, must configure the server, and have to accept that speed and quality depend heavily on the combination you choose.

This round uses **Qwen3.8 Flash-Next**, an open model family that showed it could act as a coding agent in earlier tests in this project. It uses a *mixture of experts* (MoE) architecture: although it has many parameters in total, only some experts are activated for each token. This reduces computation per token, but does not remove the need to store or move the weights. That is why quantization and available memory matter so much.

Quantization represents model weights with fewer bits. The model file usually gets smaller and inference can get faster, but precision may be lost. Q2_0 is more compact; IQ3_S uses more bits; UD-Q4_K_XL uses a larger quantization still. There is no universal answer to “which one is best”: it depends on how much fits on the GPU, how the engine places the data, and whether reduced precision affects the task at hand.

I served these packs with **Basalt**, a local inference engine derived from Strata and optimized for NVIDIA Blackwell GPUs. It loads the model, manages what stays in VRAM, and exposes an OpenAI-compatible API so applications such as OpenCode can talk to it. The original article had already used Basalt on an RTX 5090. Here I kept the engine and changed the missing piece: an RTX PRO 6000 with 96 GB of VRAM, enough to test the Q4 pack as well.

The [local LLM benchmark by Fabio Akita](https://akitaonrails.com/2026/10/27/llm-benchmark-melhores-llms-open-source-hardware-caseiro/) compared Qwen3.8 Flash-Next on an RTX 5090 and a Strix Halo system. The practical takeaway was interesting: with Basalt, Q2_0 came close to 400 tokens per second on the 5090 and passed the difficult coding test. But one question remained: what happens with larger versions on a GPU with enough memory?

I have an RTX PRO 6000 Blackwell with 96 GB of VRAM. So I prepared the Basalt configurations, used the same tests from the repository, and ran Q2_0, IQ3_S, and UD-Q4_K_XL. The goal was not to invent a new yardstick, but to fill in the hardware and quantization gap left open by the original article.

## What I tested

I used three Basalt packs of Qwen3.8 Flash-Next:

| Quantization | What to expect |
|---|---|
| Q2_0 | Smallest and most aggressive; tends to favor speed and leave more room for the rest of the model. |
| IQ3_S | Slightly larger, as an option with more precision. |
| UD-Q4_K_XL | The largest of the three; uses more memory and should, in principle, preserve more information from the weights. |

The server was Basalt commit `2da2d3af`, with the GPU power limit set to 450 W. I used a maximum context of 131,072 tokens, an `int8` KV cache, and a 4.5 GiB reserve. The card reported 97,887 MiB of total memory.

The benchmark has two independent parts: a short speed test and a multi-step coding test. The speed test talks directly to the model server. The coding test uses OpenCode as an agent. This matters because the OpenCode version does not affect speed measurements, but can affect coding-test duration.

## The speed test

The runner sends three requests, each asking for up to 256 tokens:

1. a new prompt of roughly 4,000 tokens;
2. a new long prompt, called 32k in the test, that ended up with 42,400 tokens after tokenization and formatting;
3. a continuation of that long conversation to measure reuse of the KV cache.

At each stage it records input tokens per second, generated tokens per second, and time to first token. The process also tracks free VRAM, GPU utilization and temperature, power, and system memory. “Prompt tok/s” measures reading the context; “output tok/s” measures sequential generation after the model starts answering. These are different phases, so both are useful.

### Speed results

| Quantization | Generation on short prompt | Long-prompt processing | Generation on long prompt | Time to first token on long prompt | Lowest free VRAM |
|---|---:|---:|---:|---:|---:|
| Q2_0 | 483 tok/s | 9,176 tok/s | 471 tok/s | 4.62 s | 56.3 GiB |
| IQ3_S | 489 tok/s | 8,884 tok/s | 441 tok/s | 4.77 s | 40.1 GiB |
| UD-Q4_K_XL | 319 tok/s | 8,294 tok/s | 329 tok/s | 5.11 s | 15.1 GiB |

Q2_0 was fastest at generating against the long prompt, at 471 tokens per second. IQ3_S reached 441, and Q4 reached 329. The difference between Q2_0 and Q4 is clear while the model writes, but time to start answering was fairly close: 4.62 to 5.11 seconds for this 42,400-token prompt.

The short test had an interesting result: IQ3_S reached 489 tokens per second, slightly above Q2_0 at 483. On short workloads, small differences can reflect normal test variation; I would not call a winner based on those six tokens per second. Q2_0 led by a wider margin on the long prompt.

With cache reuse, the models generated at 479 tok/s (Q2_0), 434 tok/s (IQ3_S), and 346 tok/s (Q4). The first token arrived in 0.06–0.07 seconds. The prompt-processing rate looks enormous because the conversation reuses tokens that were already processed. This is not a fair comparison with a cold prompt; it shows how cache can reduce waiting during an ongoing conversation.

Memory use changes substantially. Q2_0 kept 56.3 GiB free at its lowest point, IQ3_S kept 40.1 GiB, and Q4 kept 15.1 GiB. Q4 fit and worked without using all the card's memory, but left much less room for additional models, parallel sessions, or other GPU processes.

### More VRAM does not mean three times the speed

The PRO 6000 has three times the memory capacity of the 5090, but capacity and speed are different resources. The most obvious gain here was being able to load the Q4 quantization and still have operational headroom. It also leaves room to keep more experts and context on the GPU without relying as much on system RAM.

But these results do not support a rule like “a card with three times the memory makes the model three times faster.” Prompt processing benefits from parallel computation; token-by-token generation tends to depend more on how much data the model must move at each step and the available bandwidth. Quantization changes that balance too. That is why I measured prefill and decode separately.

Q2_0 processed the long prompt at 9,176 tokens per second and Q4 at 8,294. The prefill gap was smaller than the decode gap: Q2_0 generated 471 tok/s versus 329 for Q4. Time to first token remained in the same general range. If your workload is a conversation with long answers, decode matters a lot. If an agent receives a repository or long history at every step, prompt processing and cache can matter just as much.

It is also useful to distinguish “context capacity” from “speed at that context length.” The server was configured for up to 131,000 tokens, but that does not mean a prompt of that size will be processed quickly or that model quality remains the same at the limit. Our cold test had 42,400 effective tokens. That is long enough to show prefill differences, but it does not validate 100,000-token sessions.

### Comparison with the 5090

The original article measured Q2_0 and IQ3_S on Basalt with an RTX 5090. There, Q2_0 generated at 377–400 tok/s and processed the long prompt at 8,573 tok/s; IQ3_S reached 223–231 tok/s and 8,196 tok/s, respectively. On the PRO 6000, we recorded 471/9,176 for Q2_0 and 441/8,884 for IQ3_S.

| Quantization | RTX 5090 with Basalt, original article | RTX PRO 6000 with Basalt, this run |
|---|---:|---:|
| Q2_0, long-prompt generation | 377–400 tok/s | 471 tok/s |
| Q2_0, long-prompt processing | 8,573 tok/s | 9,176 tok/s |
| IQ3_S, long-prompt generation | 223–231 tok/s | 441 tok/s |
| IQ3_S, long-prompt processing | 8,196 tok/s | 8,884 tok/s |

These numbers provide context, but this is not a perfectly controlled A/B test. The cards are different, and the runs used different commits and pack configurations. We can safely state the result measured on each machine; we cannot attribute all of the difference to the GPU alone or expect it to repeat with every Basalt version. The IQ3_S result in particular is a large jump that should be repeated under identical conditions before drawing a general conclusion.

The Q4 result is the most direct addition: in the original article it did not complete the quality tests on the 5090 or Strix system. Here, on the PRO 6000, it ran all three speed canaries and all three hard-canary rounds. That answers whether it fits and works on this machine. It does not answer whether Q4 is better for every project.

### What I did not measure

The numbers are from one request at a time. I did not test multiple simultaneous conversations, so I do not know the server's aggregate throughput or how per-user latency would change with several sessions. I also did not measure the entire workstation's wall power or cost per token; the runner collects GPU-reported power, but this run is not an energy-efficiency analysis.

The generation test asks for a short, standardized answer. This makes comparison between quantizations easier, but does not reproduce every aspect of a real agent: tool calls, long answers, different attachments, project-specific prompts, or idle periods. The hard canary covers some agent behavior, but uses one coding task and a small codebase.

## The coding test

The hard canary is more than asking the model to write a function. It gets a small Python payment-reconciliation project and must develop it over a continuous four-turn OpenCode session. The project uses Python's standard library, `Decimal` for money, and `unittest`; there is no web framework or external dependency.

In the first turn, the model implements a specification covering charges, refunds, fees, currencies, dates, and time windows. Some rules are buried in a long history that includes rejected suggestions and a later correction. Among the rules: account IDs are case-insensitive; test events are ignored but reserve their IDs; fees must be in USD; and internal `house_` accounts are excluded from the grand total.

The requests then change and accumulate:

- refunds must find a charge even when they appear before it in the file; limits are evaluated in time order;
- chargeback events are added, with their own rules and interactions with refunds;
- the CLI gains an option to display balances in another currency;
- currency conversion is extracted into a separate module;
- the model must add regression tests, preserve the supplied tests, and review the code against every earlier requirement.

At the end, 36 hidden tests check the result. The model cannot see them while working: the code runs in a network-disabled sandbox with limited project access. The hidden tests are mounted only during evaluation.

### Hard-canary results

Each quantization was run three times. All nine runs passed the 36 hidden tests.

| Quantization | Scores | Average time for four turns |
|---|---|---:|
| Q2_0 | 100, 100, 100 | 5.2 min |
| IQ3_S | 100, 96, 100 | 5.8 min |
| UD-Q4_K_XL | 100, 100, 100 | 12.5 min |

The 96 on the second IQ3_S run was not a functional failure: all 36 hidden tests passed. The dependency auditor mistook imports from the local `ledger` package for an external dependency and deducted four points. The other two IQ3_S runs did not trigger that warning.

Q4 took more than twice as long as Q2_0 on average to finish the task, despite passing the same tests. That does not necessarily mean it “thought” for longer: wall time includes generation, tool calls, test execution, and agent decisions. The model also produces different amounts of text across runs. Still, the practical result is clear: on this task, Q4 showed no quality advantage that offset its lower speed.

There is a comparison caveat. The original article used OpenCode 2.1.4; these PRO 6000 runs used OpenCode 2.2.0 because it was the available version. Coding-test durations should therefore not be compared directly with the original article. Speed measurements are independent of OpenCode.

## What these numbers do and do not say

For this model, machine, Basalt version, and these prompts, Q2_0 was the fastest on the long prompt and completed the hard canary sooner. IQ3_S was close in speed and also passed every test. Q4 worked well on the PRO 6000, where it fit with operational headroom, but it was slower and showed no quality gain in this test.

This does not prove that Q2 is always enough or that Q4 is useless. The test covers a small Python project, with long context and changing requirements; it does not cover a large repository, context above 100,000 tokens, other kinds of tasks, or several people using the server at once. There were only three runs per quantization. For a purchase decision or a specific workflow, I would still run tasks representative of my own work.

The PRO 6000 adds a simple piece of evidence to the discussion: the Q4 pack that did not fit in the original article's home hardware could be measured on a 96 GB card. It delivers 329 tokens per second on the long prompt, while Q2_0 reaches 471 and passed the same coding tests. More VRAM opens room to try larger models; it does not guarantee that the larger quantization is the best choice.

I would start with Q2_0 for a local agent in this setup because it was fast and missed none of the tests in this sample. If a difficult-to-reproduce error appears, or the job involves much longer context, IQ3_S is a reasonable next step: it costs speed compared with Q2_0, but stayed close in the benchmark and also passed the tests. I would choose Q4 when there is a concrete reason for that larger quantization — for example, my own evaluation shows it improves results on my code — rather than simply because the card can load it.

## In summary

- **Q2_0:** fastest on the long prompt at 471 tok/s; 5.2 minutes average on the hard canary.
- **IQ3_S:** 441 tok/s and 5.8 minutes average; passed all 36 tests on every run.
- **UD-Q4_K_XL:** 329 tok/s and 12.5 minutes average; passed all 36 tests and had at least 15.1 GiB of VRAM free.

Configuration files and detailed results are in the [benchmark repository](https://github.com/akitaonrails/llm-opensource-benchmark), and the changes were proposed in [PR #1](https://github.com/akitaonrails/llm-opensource-benchmark/pull/1). The original article remains the reference for results on the RTX 5090 and Strix Halo; this post documents the complementary RTX PRO 6000 run.
