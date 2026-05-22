---
title: "Theorem Proving and Proof Repair With Large Language Models"
excerpt: "Fine-tuned LLMs to repair broken proofs in the Lean 4 theorem prover, achieving 10+ percentage point improvement over baselines. MSc AI dissertation at Imperial College London."
collection: portfolio
date: 2024-09-01
---

**Duration:** May 2024 – Sep 2024  
**Repository:** [github.com/monkeytim19/proof-repair-LLM-Lean4](https://github.com/monkeytim19/proof-repair-LLM-Lean4)  
**Supervised by:** Dr. Pedro Mediano, Imperial College London

MSc in AI individual research project exploring the use of Large Language Models for theorem proving and proof repair in the Lean 4 theorem prover.

- Developed a pipeline to curate proof breakage data by scraping Git repositories of Lean 4 projects
- Created the *mathlib4-repair* dataset containing 8,876 data points by scraping the mathlib4 library
- Fine-tuned ByT5-Small and DeepSeek-Prover LLMs on the dataset using LoRA and repaired over 10 percentage points more proofs than the baselines
- Performed a literature review on AI-based theorem proving and wrote a full project report

**Tools:** Python, PyTorch, HuggingFace, Lean 4, Linux, Shell (Bash/Zsh), Slurm
