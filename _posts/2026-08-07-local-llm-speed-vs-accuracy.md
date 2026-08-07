---
title: "I Benchmarked Qwen's Fast MoE Model Locally. It Was Confidently Wrong Half the Time."
date: 2026-08-07 07:30:00 -0400
categories: [blog]
tags: [ai, experiments, llm]
excerpt: "The MoE variant is 3.5x faster than its dense sibling — but on my local SWE-bench run it submitted wrong or invalid answers over 52% of the time it 'completed' a task. Here's what I found."
header:
  og_image: /assets/images/2026-08-03-ogimage.jpeg
---

[![path](https://images.unsplash.com/photo-1480350376518-4575ee35bf49?q=80&w=2070&auto=format&fit=crop)](https://unsplash.com/photos/a-close-up-of-a-wall-with-lines-on-it-O-8Fmpx7HqQ)

Qwen's MoE model `Qwen 3.6 35B A3B` generates tokens **3.5x faster** than its dense 27B sibling on my local hardware. Qwen's published benchmarks suggest you only give up 5–10% accuracy for that speedup. That's an incredible trade-off.

I ran a local SWE-bench experiment to find out if I could replicate the numbers. The headline result: when the MoE model _did_ submit an answer, **it was wrong or completely invalid over 52% of the time**.

Here's how I got there.

---

## Why Speed Matters for Local LLMs

With hosted models, your limiting resource is money — price per token caps how much you can do. With local models, tokens are essentially free (ignoring cost of electricity). So the constraint shifts: _time_ becomes precious. Tokens per second dictate how fast a model can solve problems, and on commodity hardware that can feel like a crawl compared to hosted alternatives.

This is what makes Mixture of Experts (MoE) architectures appealing. In an MoE model, only a subset of parameters are active for any given token, making generation significantly faster than their dense model counterparts. `Qwen 3.6 35B A3B` has 35 billion total parameters but only activates ~3 billion at a time — hence "A3B".

![qwen-tokens-pers-second-comparison](/assets/images/localllm/token-perf-comparison.svg)

The chart above shows the results of running a prompt with reasoning disabled purely for the purpose of capturing token generation speed. Accuracy (or even coherence) was not a factor. Running both models locally on an RTX 5090 using llama.cpp server:

```sh
llama-server -m ./models/qwen3.6-27b-q4_k_m.gguf \
  -ngl 99 \
  -c 80000 \
  --port 8080 \
  --host 0.0.0.0 \
  -fa on \
  --log-file logs/$(date +%s).log
```

and then running the prompt requests from another machine on my network:

```sh
curl http://192.168.1.173:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {
        "role": "user",
        "content": "Write an extremely detailed, exhaustive tome about the history of computer science starting with invention of fire. It should take a human lifetime to read."
      }
    ],
    "stream": true,
    "max_tokens": 5000,
    "ignore_eos": true,
    "chat_template_kwargs": {
      "enable_thinking": false
    }
  }'
```

We can see that the 35 billion parameter MoE variant is about 3.5x faster at generating output than the dense 27 billion parameter model. We get about 250 tokens per second with `35B A3B` vs about 72 tokens per second with `27B`.

So if our limiting factor is time when running local LLMs, then we should always use this MoE variant!

Not so fast.

## Time vs Accuracy

As with all things there is a trade off. The MoE variant isn't quite as 'smart' as the denser model. Comparing the published benchmarks we see that the performance looks comparable but the lead is clearly with the 27B variant.

Interestingly, on their [HuggingFace Model Card](https://huggingface.co/Qwen/Qwen3.6-27B#qwen36-highlights), their SWE-Bench Verified results show only about a 5-10% difference in task completion. Similar results are shown across other common benchmarks as well.

![HF Bench Results](/assets/images/localllm/qwen3.6-hf-card-bench-results.png)

This is an extremely impressive result and I wanted to verify it by running the benchmark locally. If I can get 3.5x the speed to similarly accurate results, then I'm all for it!

## Local Benchmarks

I set off to run the benchmark locally. I cloned the [SWE-bench](https://github.com/swe-bench/SWE-bench) and [mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent) repositories and got everything running... And I immediately realized that this was not as simple as I expected. Running these LLM benchmarks is not like running a benchmark against a typical algorithm. It's slow! (particularly on commodity hardware.) To give you an idea of just how slow, the average time to completion I was seeing for a single task was about 15-25 minutes. The SWE bench verified tasks (which is only a subset of the full SWE bench set) contains 500 tasks. So at 15 minutes per task, the full benchmark would take over 5 days of my GPU cranking 24 hours a day. So I settled on the 'Lite' subset, which is designed to be simpler, allowing for faster completion.

Here, faster is relative. I didn't want to wait 24 hours for a result so I took a subset of the subset, the first 50 tasks in SWE Bench Lite. I also limited the allowed time per task to only 5 minutes. I figured this would level the playing field between the MoE model and the dense model and also save time on the data collection.

```yaml
# mini.yaml - configuration overrides allowing me to hit my local LLM
model:
  model_name: "openai/qwen3.6-2b"
  cost_tracking: "ignore_errors"
  model_kwargs:
    api_base: "http://192.168.1.173:8080/v1"
    api_key: "not-needed"
    drop_params: true

agent:
  wall_time_limit_seconds: 300 # 5 minutes
```

```sh
# Example of running through the tasks
mini-extra swebench \
  --config /path/to/default/swebench.yaml \
  --config mini.yaml \
  --output bench-27b \
  --subset lite \
  --split test \
  --slice ":10" \ # I ran the tests in increments of 10 tests so I could check the progress
  --workers 1
```

### Results

Here are the results of my local benchmark:

![SWE bench comparison](/assets/images/localllm/swe-bench-chart-1.svg)

> **<i class="fas fa-triangle-exclamation"></i> Note:**  
> The other errors category comprises `Timeout`, `RepeatedFormatError`, `ContextWindowExceededError`
>
> Qwen 3.6 27B:
>
> - 17 Timeouts
>
> Qwen 3.6 35B-A3B
>
> - 4 Timeouts
>
> {: .notice--warning}

### Some Thoughts

My absolute numbers are lower than Qwen's published results, which isn't surprising. The agent harness matters — I used the default mini-swe-agent while Qwen almost certainly used a more optimised one. Q4_K_M quantization also has a cost. And my aggressive 5-minute time limit hurt the dense model more, since it tends to think longer before acting.

That said, the **gap between the two models** is what I'm most interested in, and here the results roughly hold: the dense 27B model completed only 2 more tasks than the MoE variant, which is consistent with Qwen's claimed 5–10% difference.

But task completion rate alone isn't the full story.

### Confidently Wrong

What jumps out is not how many tasks the MoE model failed to complete — it's how often it completed a task _incorrectly_ and submitted anyway.

- **37.5% of submissions** contained code that did not fix the issue (15 out of 40)
- **15% of submissions** were invalid patches — the evaluator couldn't even apply them; the model had hallucinated a diff from scratch
- Combined: **the MoE model was confidently wrong 52.5% of the time it produced an answer**

> **<i class="fas fa-triangle-exclamation"></i> Caveat:**  
> Because the 27B model timed out more frequently under the 5-minute limit, I may not have given it enough rope to hallucinate at the same rate. For a fair apples-to-apples comparison, both models should be allowed to run to completion. Take the accuracy comparison with a grain of salt.
> {: .notice--warning}

## Takeaways

None of this is meant to discourage you from using Qwen 3.6 — I use the dense 27B variant as my daily driver and it's excellent. The point is more nuanced:

**On model selection:** The MoE variant is the right tool for high-volume, low-stakes tasks where you'll review the output anyway — scaffolding, boilerplate, first drafts. For agentic tasks where the model submits code autonomously and you want it to be right, dense models may be more appropriate.

**On benchmarks:** Published numbers are a starting point, not a verdict. Quantization, agent harness, time limits, and task distribution all affect results significantly. Particularly, for local LLMs, run your own experiments on your own hardware before making architectural decisions, but bring your patience.

**On confidence:** This is the one that isn't talked about as much when labs post their scores. A model that submits a plausible-looking but broken patch has _failed invisibly_. I found that the MoE variant did this frequently. That's the real risk of leaning on these local models for autonomous work. Right now they are still just tools that require human oversight, albeit extremly powerful, almost magical tools.
