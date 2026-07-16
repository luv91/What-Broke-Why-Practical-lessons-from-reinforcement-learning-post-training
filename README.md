# What Broke & Why

### Practical Lessons from Reinforcement Learning Post-Training

**Luv Verma, PhD** · First edition, v1.0 · July 2026 · 648 pages · 157 chapters

[![DOI](https://img.shields.io/badge/DOI-10.5281%2Fzenodo.21115798-4FB286?style=flat-square)](https://doi.org/10.5281/zenodo.21115798)
[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey?style=flat-square)](https://creativecommons.org/licenses/by/4.0/)
[![Pages](https://img.shields.io/badge/pages-648-6B7A80?style=flat-square)](#)

**[Read the book — free PDF →](https://doi.org/10.5281/zenodo.21115798)**

---

## Why this book exists

The night the search agent's entropy collapsed at step 30, I had eight papers open in eight tabs and a training run dying in the ninth.

The literature on reinforcement learning is vast, and almost none of it is about the run in front of you. The classic textbook is a different kind of RL. The alignment books are about a different judge — a learned reward model, not a program. The papers each hold one piece, measured on a model ten times the size of mine, on a cluster I did not have, with the failures edited out.

Nothing sat at the intersection of all three constraints:

| Constraint | This book | The existing shelf |
|---|---|---|
| **Lane** | RL with verifiable rewards — a unit test, exact match, an F1 score | Mostly preference tuning against a learned judge |
| **Budget** | 1–8 GPUs, start to finish (A100-40GB, H100-80GB) | Frontier scale: 32B base, dozens of cards, two weeks |
| **Stance** | Organized around runs that broke, every number with a manifest path | Clean introductions, tutorials, surveys |

So I wrote down the run.

---

## How the book works

Three layers. Read the **Journey** once, in order. Use the **Science** to see what the runs proved. Keep the **Reference** open when your own run breaks.

```
JOURNEY                  SCIENCE                 REFERENCE
Parts I–VI · Ch 1–69     Part VII · Ch 70–76     Parts VIII–XV · Ch 77–157

the runs we did:         what the runs proved:   rewards, evals, architecture,
math, search, entropy,   sharpening, fork        failures, repro, rollout,
MoE, SWE, distillation   tokens, weight movement hardware

read once, in order      read after the runs     return when a run breaks
```

---

## The journey: five programs, five pivots

Each program ended with a failure that forced the next one.

| # | Program | Part | It broke because | The pivot |
|---|---|---|---|---|
| 01 | Math & GSM8K | II · Ch 6–24 | reward pinned at 0 from cold start | partial credit + curriculum |
| 02 | Search Agent | III · Ch 25–37 | multi-reward got hacked | correctness-gated reward |
| 03 | Entropy & Clip-Cov | IV · Ch 38–48 | entropy collapsed, then exploded | Clip-Cov |
| 04 | MoE & GSPO | V · Ch 49–59 | routing drift, ratio fragility | sequence-level importance sampling |
| 05 | SWE-bench → OPD | VI · Ch 60–69 | sparse reward hit a capacity wall | on-policy distillation |

---

## The symptom router

When your run breaks, don't search the table of contents. Read the signal, run the first check, and go.

| Your dashboard shows | First check | Go to |
|---|---|---|
| `reward = 0` | group variance, extractor, format reward | Ch 11–13 · Cold Start / Partial Credit |
| `reward up, eval down` | reward components, behavioral audit | Ch 31–32 · Correctness Gate |
| `entropy collapses, then explodes` | entropy, reward_std, length | Part IV · Clip-Cov (Ch 42–43) |
| `solve-rate flat, reward nonzero` | is the task reachable? reward sparsity | Ch 65–66 · OPD Recipe |
| `MoE ratios spike, routing drifts` | ratio_max, routing entropy, expert load | Ch 51–54 · GSPO / Routing |
| `eval changed, won't reproduce` | sample size, decoding, extractor | Part IX · Evaluation Discipline |
| `outputs changed after speedup` | KL to BF16 rollout, router flips | Ch 130, 134 · Rollout Contract |
| `OOM, or only one GPU` | resident weights, KV cache, sharding | Part XV · Hardware |

---

## Three decisions the book makes for you

Each has a measurement at the root and a threshold at every branch.

### 1. Should I be running RL at all?

Measure base success on your task — pass@1, before any training, on a real eval. This one number decides everything downstream.

```
                    base success on the task?
                              │
        ┌─────────────────────┼─────────────────────┐
       ~0%                 20–80%                 ~100%
        │                     │                     │
   too hard              RL HELPS               saturated
   no variance,          informative            nothing left
   no gradient           advantage              to sharpen
        │                     │                     │
   SFT cold-start        → Part VIII           eval only,
   or change task          reward design       find a harder eval
   or distill (Ch 68)
   Ch 11–13              Table M.1             Table M.1
```

> **No variance, no RL.** RL sharpens capabilities the base model can already sometimes exhibit — it does not add new ones. That's why the band is 20–80%.

### 2. My reward is lying to me

```
              reward curve vs. held-out eval?
                              │
        ┌─────────────────────┼─────────────────────┐
   reward = 0           reward ↑, eval ↓        reward ↑, eval flat
        │                     │                     │
   COLD START           REWARD HACKING          PLATEAU
   extractor? binary    auxiliaries bought      correctness-only
   reward? identical    without correctness     has no gradient left
   rollouts?                  │                     │
        │               correctness-gate       graded reward (F1)
   partial credit       the auxiliaries        + gated channels
   + format reward
   Ch 11–13             Ch 31–32               Ch 26–30
```

The gate is the SiLU/Swish term `f1 · σ(f1)` — smooth, not a hard 0/1 flag, because a hard gate fights the optimizer:

| F1 | gate value | state |
|---|---|---|
| 0.0 | 0.000 | closed |
| 0.1 | 0.052 | barely open |
| 0.3 | 0.172 | weak |
| 0.5 | 0.311 | moderate |
| 0.7 | 0.468 | strong |
| 1.0 | 0.731 | near-full |

> **Auxiliary rewards must be correctness-gated.** Wrong answer → gate closed → behavioral points worth nothing, no matter how grounded or efficient. The road to higher reward now runs through being right.

### 3. Is this speedup actually free?

Serving asks *is it faster?* RL must also ask: *is the policy that generated these rollouts still the policy I'm training?*

```
                  classify the acceleration
                              │
        ┌─────────────────────┼─────────────────────┐
  safe-by-construction      exact                 lossy
  same computation      bit-identical        quantized KV/weights
        │                     │                     │
    Gate B only          Gate B + verify      ALL THREE GATES
                         outputs identical     A, B, and C
        │                     │                     │
  spec decoding         prefix caching        FP8-KV died here
  Ch 137                Ch 139                Ch 140
```

| Gate | What it checks | Bar |
|---|---|---|
| **A** | mean per-token forward KL, matched seed/T, generated tokens only | **< 0.05 nats/token** |
| **B** | real end-to-end step wall-clock — not a microbenchmark | must actually be faster |
| **C** | 500-sample held-out eval after a short RL run | must hold |

**The flagship failure — FP8-KV on Qwen3-14B:** KL of **7.61** against a bar of 0.05 — a failure by **152×**, with **~40% of tokens disagreeing** at matched seed and T=0. And it was **0.97×** — 3% *slower*. There was no point running Gate C.

> **A serving-style "97% accuracy recovery" is a FAILING Gate A.** FP8 weights are fine — quantized once at load, a fixed perturbation the model trains around. FP8 KV is poison: the cache is re-read lossily on every attention op, so error compounds across the sequence.

---

## The doctrine

Eight failures, eight rules. Each one replaced a belief.

1. **No variance, no RL.** — Ch 11–13
2. **Auxiliary rewards must be correctness-gated.** — Ch 31–32
3. **Numbers need a canonical eval.** — Ch 37, 85
4. **Entropy needs damped feedback, not a global push.** — Ch 42–43
5. **Sequence-level ratios are the safe default at scale.** — Ch 51–54
6. **If RL can't reach reward, change the signal.** — Ch 65–66
7. **Speedups need an inference-correctness contract.** — Ch 130, 134
8. **An implementation wall is not an algorithmic law.** — Ch 152

---

## Who it's for

- You are running RL post-training on one to eight GPUs, and it is not working.
- You want the algorithms — GRPO, DAPO, GSPO, Clip-Cov, on-policy distillation — with the operating knowledge attached.
- You have been burned by a training-eval peak that didn't survive an honest eval.

**What it isn't:** not a from-scratch tutorial, not a survey, not about RLHF or DPO against a learned judge, and not frontier-scale.

---

## Cite

```bibtex
@book{verma2026whatbroke,
  author    = {Verma, Luv},
  title     = {What Broke \& Why: Practical Lessons from
               Reinforcement Learning Post-Training},
  edition   = {First},
  version   = {1.0},
  year      = {2026},
  publisher = {Zenodo},
  doi       = {10.5281/zenodo.21115798},
  url       = {https://doi.org/10.5281/zenodo.21115798}
}
```

> Verma, Luv. *What Broke & Why: Practical Lessons from Reinforcement Learning Post-Training.* First edition, version 1.0. Zenodo, 2026. DOI: 10.5281/zenodo.21115798.

---

## License

© 2026 Luv Verma. Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Share and adapt for any purpose, with credit.
