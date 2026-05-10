---
name: vla-policies-overview
description: Use when picking, comparing, or migrating between VLA / imitation-learning policies (OpenVLA, Octo, Pi0, RDT2, GR00T, ACT, Diffusion Policy, VQ-BeT). Surfaces what each policy actually is, what it expects in/out, and what it's good at — so the choice is deliberate, not "the one I read about last".
---

Quick map of the major VLA / IL policy families. Use as a decision aid; for implementation conventions see `lerobot-conventions`.

## At a glance

| Policy | Type | Action chunking | Vision encoder | Backbone | Best at | Watch out for |
|---|---|---|---|---|---|---|
| **OpenVLA** | VLA (token regression) | No (single-step) | DINOv2 + SigLIP | Llama-2 7B | OOD generalization, language conditioning | Slow inference (~5 Hz on consumer GPU); single-arm 7-DoF default |
| **Octo** | Foundation transformer | Yes | T5 + ViT | Diffusion head | Multi-task pretraining (OXE) | Pretrained data quality limits ceiling |
| **Pi0 / Pi0-FAST** | Flow-matching | Yes | PaliGemma | Action experts | SOTA dexterity, bimanual | Compute-heavy; less reproducible OSS than ACT |
| **RDT2** | Diffusion transformer | Yes | DINOv2 | Diffusion | Bimanual, dual-arm | Newer; ecosystem still maturing |
| **GR00T (N1, N2)** | VLA | Yes | NVILA-derived | Action expert | Humanoid native, NVIDIA stack | Tied to Isaac / NVIDIA stack; bring-up effort |
| **ACT** | Transformer encoder-decoder | Yes (chunk_size) | ResNet18 | Small CVAE | Fast, lightweight, well-documented | Less generalization than VLA-class |
| **Diffusion Policy** | Diffusion (UNet or Transformer) | Yes (horizon) | ResNet18 | UNet / Transformer | Multi-modal action distributions | Slower than ACT; needs care with EMA |
| **VQ-BeT** | Discrete-action transformer | Yes | ResNet | Mini-GPT | Multi-modal demos | Tokenizer needs tuning per dataset |

## How they differ in practice

**Action format.**
- OpenVLA: 7-DoF token (de-tokenize → continuous). Single timestep.
- ACT / Diffusion / VQ-BeT / Pi0 / RDT2 / GR00T / Octo: continuous chunks (predict N future actions, execute first K).

**Compute at training time.**
- ACT, Diffusion Policy, VQ-BeT: train on one A100 in hours.
- Octo, RDT2: multi-GPU days for pretraining; fine-tune cheap.
- OpenVLA, Pi0, GR00T: serious compute (multi-node) for pretraining; LoRA / fine-tune feasible on single 80 GB GPU.

**Compute at inference.**
- ACT, VQ-BeT, Diffusion Policy: real-time on mid-tier GPU.
- OpenVLA: slow; needs quantization or batched inference for ≥10 Hz.
- Pi0 / RDT2 / GR00T: depends on action-expert size; usually OK with action chunking.

**Language conditioning.**
- OpenVLA, Octo, Pi0, RDT2, GR00T — yes (different mechanisms).
- ACT, Diffusion Policy, VQ-BeT — no (task-conditioning via separate goal tokens or single-task assumption).

## Picking one — quick decision tree

1. **Single task, single arm, fast iteration?** ACT or Diffusion Policy. ACT first if you want speed; Diffusion if your demos are multi-modal.
2. **Single task, want best success rate?** Diffusion Policy.
3. **Multi-task pretraining, want to fine-tune?** Octo (OSS, OXE pretraining) or Pi0 (if you have access).
4. **Bimanual / dual-arm?** RDT2 or Pi0. ACT works too but lower ceiling.
5. **Humanoid?** GR00T (NVIDIA stack) or Pi0 with humanoid action head. Roll your own at your peril.
6. **Want OOD language generalization?** OpenVLA. Accept slow inference.
7. **You are doing research and need a clean baseline?** ACT.

## Common confusion points

- **"Diffusion policy" can mean three things:** (a) Cheng Chi's original 2023 paper (UNet over actions), (b) the LeRobot `DiffusionPolicy` class (UNet variant by default), (c) any policy whose action head uses a diffusion process (Octo, RDT2). Specify.
- **"VLA" is loose.** Originally meant "vision-language-action" (RT-2, OpenVLA — single-step, language-conditioned). Now used for any VLM-derived policy regardless of action format. Pi0 / Octo / GR00T are VLA-adjacent in this loose sense.
- **OpenVLA's 7-DoF default is a constraint.** Bimanual or non-7-DoF embodiments need either re-tokenization or a different policy.
- **GR00T runs on NVIDIA stack.** Isaac Sim / Lab + Triton for serving. If you don't already use NVIDIA infra, the bring-up cost dominates.
- **Action chunking ≠ predicting the future correctly.** Chunk size is a robustness mechanism (decouple inference rate from control rate), not a guarantee of long-horizon behavior. Long-horizon needs language or hierarchical policies.

## When you also need

- `lerobot-conventions` — for the dataset/training plumbing of any of these.
- `dataset-versioning` — before training one.
- `policy-eval-protocol` — before claiming the chosen policy works.
- `humanoid-wbc` — if picking GR00T or Pi0 for a humanoid.
