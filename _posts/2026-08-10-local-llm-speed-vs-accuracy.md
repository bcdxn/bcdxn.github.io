---
title: "Local LLMs Speed vs Accuracy"
date: 2026-08-04 07:30:00 -0400
categories: [blog]
tags: [ai, experiments, llm]
excerpt: ""
header:
  og_image: /assets/images/2026-08-03-ogimage.jpeg
---

[![path](https://images.unsplash.com/photo-1480350376518-4575ee35bf49?q=80&w=2070&auto=format&fit=crop)](https://unsplash.com/photos/a-close-up-of-a-wall-with-lines-on-it-O-8Fmpx7HqQ)

In both hosted models, and local models your limiting factor boils down to the same precious resource - tokens. But how we reach that limit is very different. With hosted models, price per token and ultimately the total amount of money you want to spend on a particular task is the finite resource. With local models, tokens are essentially free (not counting the cost of electricity and other externalities) -- so money isn't the limiting factor. Instead, _time_ becomes precious. Tokens per second dictate how fast a model can generate output and therefore solve problems and on local commodity hardware that can slow to what feels like a crawl (compared to hosted alternatives).

As a result new architectures specifically designed for efficiency have been developed -- enter Qwen's Mixture of Experts (MoE) variants. In MoE architectures, only a subset of the full model parameters are active when generating tokens making them faster. In the case of `Qwen 3.6 35B A3B` it is _significantly_ faster than its dense 27 billion parameter sibling `Qwen3.6 27B`.

![qwen-tokens-pers-second-comparison](/assets/images/localllm/token-perf-comparison.svg)

The chart above shows the results of running a prompt with reasoning disabled purely for the purpose of capturing token generation speed. accuracy (or even coherance) was not a factor. Running both models locally on an RTX 5090 using llama.cpp server:

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

We can see that the 35 billion parameter MoE variant is about 3.5x faster at generating output than the dense 27 billion parameter model. We get about 250 tokens pers second with `35B A3B` vs about 72 tokens per ssecond with `27B`.

So if our limiting factor is factor is time with running local LLMs, then we should always use this MoE variant!

Not so fast.

## Time vs Accuracy

As with all things there is a trade off. The MoE variant isn't quite as 'smart' as the denser model. Comparing the published benchmarks we see that the performance looks comparable but the lead is clearly with the 27B variant.

Interestingly, on their [HuggingFace Model Card](https://huggingface.co/Qwen/Qwen3.6-27B#qwen36-highlights), their SWE-Bench Verified results show only about a 5-10% difference in task completion. Similar results are shown across other common benchmarks as well.

![HF Bench Results](/assets/images/localllm/qwen3.6-hf-card-bench-results.png)

This is an extremely impressive result and I wanted to verify it running the benchmark locally. If I can get 3.5x the speed to similarly accurate results, then I'm all for it!

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

![SWE bench comparison](/assets/images/localllm/swe-bench-comparison.svg)

### Some Thoughts

1. My numbers seem quite a bit different than the published benchmarks. Some reasons for that:

- I'm guessing agent harness has some impact. I used the default mini-swe-agent. I assume Qwen used their own harness.
- I'm sure quantization levels (I used Q4_K_M quantized model ways) impacts performance
- As previously stated, I set aggressive time limits on task completion. Giving the model more time would probably yield better results particularly for the dense variant.

3. The dense model only completed 2 more tasks than the MoE variant which does align with the 5-10% difference quoted in Qwen's results
4. The MoE model is happy to submit wrong answers

> **<i class="fas fa-triangle-exclamation"></i>Warning:**  
> I don't want to put a whole lot of stock in comparing the accuracy numbers between the 27B runs and 35B A3B runs. Because the 27B modele timed out more frequently, I may simply have not given enough time for the dense model to hallucinate as the MoE model did. For a fair comparison, both models should be given a chance to run to completion.
> {: .notice--info}

### Confidently Wrong

The 5-10% gap in performance noted in the benchmark doesn't tell the full story. Instead, what jumps out at me is the fact that the MoE model is confidently wrong _a lot_. It is happy to submit wrong answers.

- It submitted code that did not fix the issue 15 times out of 40 submissions. **It was 'wrong' 37.5% of the time.**
- The patch was invalid 6 times, i.e. the bench evaluator couldn't even apply the patch because it was garbage. **It fully hallucinated 15% of the time**
- Combining the wrong answers and invalid diffs **it was confidently wrong 52.25% **of the time that it 'completed' the task

## Takeaways

This article isn't meant to beat up on Qwen 3.6 or to discourage its use. In fact I use the dense variant as my daily driver and only go over to Claude when I really have to. Instead, all of this is to say

1. Benchmark numbers on their own don't tell the full story (particularly when you're running quantized variants locally)
2. Don't blindly trust LLMs to write your code
