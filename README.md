# Verilog Agent Format

Fine-tuning LLMs to clean up compiler-generated Verilog using LoRA + iterative Yosys verification feedback.

## What & Why

The LiveHD hardware compiler generates verbose, hard-to-read Verilog full of redundant `_GEN_` wires and unnecessary constructs. Cleaning this up manually requires experienced hardware engineers.

This project fine-tunes code LLMs to automate that cleanup, and uses **Yosys formal equivalence checking** as the ground truth not BLEU score, not string match, but actual logical equivalence.

## Models Fine-tuned

| Model | Params | First Pass | After Refinement |
|---|---|---|---|
| Qwen3-30B | 30B | 87.3% | 88.1% |
| Devstral-Small | 24B | 84.8% | **88.4%** |
| Qwen2.5-7B | 7B | 74.6% | 86.8% |
| DeepSeek-Coder-V2-Lite | 16B | 0%* | 0% |

*DeepSeek collapsed whitespace between all tokens logically correct but unparseable by Yosys.

## How It Works

```
LiveHD Verilog (verbose) → Fine-tuned LLM → Clean Verilog → Yosys Equivalence Check
                                                                        ↓ (if fail)
                                                              Re-prompt with error log → Retry
```

1. **Dataset** — 4,914 training pairs of LiveHD-generated Verilog → Verible-formatted clean Verilog
2. **Fine-tuning** — LoRA (r=16, α=32) on attention layers, bfloat16, single GPU
3. **Evaluation** — Yosys logical equivalence checking on 614 held-out examples
4. **Iterative refinement** — Failed predictions get re-prompted with the Yosys error log for self-correction

## Key Findings

- All three working models converge to **~88% after one refinement round** regardless of starting point suggesting a hard ceiling on this test set
- Smaller models (Qwen2.5-7B) benefit more from refinement **48.1% recovery rate** vs 6.4% for Qwen3-30B
- Timeout failures are recoverable; logical mismatch failures are not
- The `signed` keyword is a shared blind spot across all models consistently dropped from memory and arithmetic circuits
- GRPO is **2.7× more token-efficient** than multi-epoch SFT for smaller models

## SFT vs GRPO Token Cost

| Model | SFT Tokens | GRPO (estimated) |
|---|---|---|
| Qwen2.5-7B (3 epochs) | 28.3M | 10.3M |
| Qwen3-30B (1 epoch) | 9.6M | 10.5M |
| Devstral (1 epoch) | 9.4M | 10.3M |

## Stack

- **Fine-tuning:** LoRA via HuggingFace PEFT, bfloat16, 4-bit NF4 quantization
- **Verification:** Yosys equivalence checking
- **Models:** Qwen2.5-7B, Qwen3-30B, DeepSeek-Coder-V2-Lite, Devstral-Small
- **Dataset:** 6,143 LiveHD Verilog pairs (4,914 train / 615 val / 614 test)

## Report

Full paper with methodology, failure analysis, and token usage breakdown → [`report.pdf`](./capstone_report.pdf)

---
