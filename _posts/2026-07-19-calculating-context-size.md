---
title: "How I Calculated the Optimal Context Size for My Local LLM"
date: 2026-07-19 07:30:00 -0400
categories: [blog]
tags: [ai, llm, ollama, qwen, local-llm]
excerpt: "A practical walkthrough of calculating context window sizing for a local Qwen model on an RTX 5090"
header:
  og_image: https://images.unsplash.com/photo-1783917053123-f68ec8e8be47?auto=format&fit=crop
---

[![path](https://images.unsplash.com/photo-1783917053123-f68ec8e8be47?auto=format&fit=crop)](https://unsplash.com/photos/delicate-white-plants-in-a-golden-misty-meadow-hY05yBvFTS8)

In my last post, I got Qwen3.6:27b running locally on my RTX 5090 via Ollama. The next question was: how big a context window can I actually give it before running out of VRAM?

I saw that Qwen3.6 can support context windows as large as 262,144 tokens, so I figured I would start with that thinking that surely the largest amount of VRAM available in a consumer GPU could handle it. I was wrong. The model would frequently slow to a crawl and/or crash with this configuration.

Context size is a hardware budget problem. The model weights take one chunk of VRAM, and the context (which stores conversation input and output) takes another. If you exceed your GPU's capacity, things crash or fall back to system RAM and grind to a halt -- this is the exact behavior I was seeing with my misconfigured model running locally.

So let's do the math and figure out what a sensible limit actually is.

## Bytes per Token

The formula used to calculate how much GPU memory is required per token during inference is:

```
bytes per token = 2 * Layers * KV Heads * Head Dim * Precision (bytes)
```

The values of the variables in this equation are dependent on the model architecture and largely specific to the self-attention mechanism implementation within the model.

| Parameter        | Definition                                                              |
| :--------------- | :---------------------------------------------------------------------- |
| 2                | Multiplier accounts for both Keys and Values; for every token processed |
| Layers           | total number of transformer layers (blocks) in the model architecture   |
| KV Heads         | The number of Key-Value attention heads per layer                       |
| Head Dim(ension) | The size of a single attention head                                     |
| Precision        | The memory size of the data type used to store the KV cache             |

We can get these details from ollama and with a bit of googling to see how the values map to the equation above.

```sh
ollama show -v qwen3.6:27b
```

This dumps the model's internal architecture. And here is how it maps

| Parameter | Value     | Source                                     |
| --------- | --------- | ------------------------------------------ |
| Layers    | ~~64~~ 16 | `qwen.block_count`                         |
| KV Heads  | 4         | `qwen35.attention.head_count_kv`           |
| Head Dim  | 256       | `qwen35.attention.key_length`              |
| Precision | ~~.5~~ 2  | using default full precision for K/V cache |

**Gotcha #1:** Qwen uses a 'hybrid' architecture that combines 16 x "standard attention layers" and 48 "gated DeltaNet layers". Standard attention layers grow linearly with each token processed; gated DeltaNet layers use constant space and so we can remove them from the calculation. So instead of the 64 that ollama gives us for the `qwen.block_count` field, we use 16 for the `Layers` variable.
{: .notice--warning}

**Gotcha #2:** The _model weights_ are quantized to 4 bits (Q4_K_M) but KV cache precision is independent and defaults to FP16. Ollama loads the weights at 4-bits to save base VRAM but allocates the KV cache at full 16-bit precision by default. It seems to be generally accepted that quantizing KV cache quickly degrades performance in coding tasks. So we'll stick with the default and instead of .5, we use 2 for the `Precision` variable.
{: .notice--warning}

So we can plug our values into the equation above:

```
bytes per token = 2 * 16 * 4 * 256 * 2
bytes per token = 65536
```

65536 bytes per token or about 64KB of VRAM per token. This is considered a very efficient model and each token is about the on-disk size of multi-page Word document. (maybe that's a weird comparison but it just strikes me how much VRAM each token requires.)

## Optimal Context Size

Total VRAM consumed by a model can be expressed as:

```
total VRAM = Model Size + (Bytes per Token * Context Size) + system overhead
```

This is just saying that the model takes its own chunk of VRAM and then we can use what's left over for the context (and basic system requirements). In my case I already know the total VRAM — 32GB. I want to solve for the Context Size variable, but first I need to know the model size.

Qwen3.6:27b is a 27 Billion parameter model which means at full precision (2 bytes / parameter) that would be:

```
27 Billion * 2 bytes = 54 Billion bytes
```

That's 54GB which is more VRAM than my 5090 has. Fortunately, as we already mentioned, Ollama provides the 4-bit quantized version by default. Q4_K_M is not a uniform 4-bit quantization — some layers are quantized to 4-bits, others to 5 or 6-bits — so we'll use an average of 5 bits:

```
27 Billion parameters * (5/8 bytes per parameter) ≈ 17 Billion bytes
```

> **Update** I realized you can also run
>
> ```sh
> ollama list
> # NAME         ID            SIZE  MODIFIED
> # qwen3.6:27b  a50eda8ed977  17GB  8 days ago
> ```
>
> So my estimate was really pretty good!

Starting with 32GB of VRAM, take away 17GB for the model weights and another 2GB for system overhead, and I'm left with 13GB for the context window. So how many tokens is that?

## VRAM Budget on an RTX 5090

| Component                | VRAM   |
| ------------------------ | ------ |
| Base model (Q4_K_M, 27B) | ~17 GB |
| System overhead          | ~2 GB  |
| Context Window           | ~13 GB |
| **Total**                | 32GB   |

```
context size (tokens) = 13 billion bytes / 65536 bytes per token
context size (tokens) ≈ 198,000
```

Most sources I see online quote a 128,000 context window as the recommendation for qwen3.6:27B on an RTX 5090, but that seems overly conservative. To give a little more margin, I'll finish the calculation assuming 10GB for the context window (I'll do more testing later to see if I can push it closer to the theoretical maximum).

```
context size (tokens) = 10 billion bytes / 65536 bytes per token
context size (tokens) ≈ 152,000

```

## Input vs. Output Context Split in VS Code

The final decision is how to allocate the tokens. Input vs output.

Before making the allocation, it's nice to understand roughly how tokens relate to words and lines of code. Qwen uses a tiktoken-based tokenizer with roughly these ratios:

| Content Type | Ratio                  |
| ------------ | ---------------------- |
| English text | ~1.3 tokens per word   |
| Code         | ~10–15 tokens per line |

So for a small 1000-line codebase, that's roughly 10000-15000 tokens.

With that in mind, I've ended up allocating significantly more tokens to the input for two reasons:

1. Larger input context gives the model a greater ability to understand an existing codebase and see "what good looks like" for a particular repository
2. I'm not trying to one-shot full applications or even full features with a local LLM. Smaller, more incremental steps are a more realistic ask.

With that in mind I've settled on the following split

| Context Type        | Tokens   | What It Covers                                                                     |
| ------------------- | -------- | ---------------------------------------------------------------------------------- |
| Input (prompt)      | ~120,000 | Open files, imports, repo structure, error traces                                  |
| Output (max tokens) | ~30,000  | Generated code responses (~2400 lines of code -- not including incremental output) |

My configuration in vscode looks like:

```json
[
  {
    "name": "ollama",
    "vendor": "customendpoint",
    "apiType": "chat-completions",
    "models": [
      {
        "id": "qwen3.6:27b",
        "name": "qwen3.6:27b",
        "url": "http://192.168.1.173:11434/v1",
        "toolCalling": true,
        "vision": true,
        "maxInputTokens": 120000,
        "maxOutputTokens": 30000
      }
    ]
  }
]
```
