# Research Collaborator: Reference Sources

Sources that directly inform specific parts of the skill.

---

## ML-Specific

### Andrej Karpathy — "A Recipe for Training Neural Networks" (2019)
- **Link**: http://karpathy.github.io/2019/04/25/recipe/
- Five-phase pipeline: data → skeleton → overfit → regularize → tune.
  Sanity checks: init loss, single-batch overfit, input-independent baseline.
- **Used in**: Execute (sequential protocol), Diagnose (sanity checks, single-batch overfit as bug detector)

### Kapoor & Narayanan — "Leakage and the Reproducibility Crisis in ML" (2023)
- **Link**: https://www.sciencedirect.com/science/article/pii/S2666389923001599
- Eight types of data leakage across 294+ studies in 17 fields.
- **Used in**: Validate (data leakage audit)

### Lipton & Steinhardt — "Troubling Trends in Machine Learning Scholarship" (2018)
- **Link**: https://arxiv.org/abs/1807.03341
- HP attribution test: vanilla LSTMs with proper tuning outperformed "innovative" architectures.
  Speculation disguised as evidence, unfair baselines, overclaiming.
- **Used in**: Validate (HP attribution test, claim audit), Language Rules

### Josh Tobin — "Troubleshooting Deep Neural Networks" (Full Stack Deep Learning, 2019)
- **Link**: http://josh-tobin.com/troubleshooting-deep-neural-networks.html
- Silent bugs: broadcasting shape mismatches, train/eval toggle, numerical instability,
  wrong loss inputs. Compare against known-good implementations to isolate problems.
- **Used in**: Diagnose (silent bugs checklist, step 1b reference implementation comparison)

### Marek Rei — "Advice for Students Doing Research Projects in ML/NLP" (2018)
- **Link**: https://www.marekrei.com/blog/ml-nlp-research-project-advice/
- Feed labels as input to test network connectivity. Reimplement SOTA first to
  validate eval setup before building anything new.
- **Used in**: Diagnose (labels-as-input sanity check, reference implementation)

### Gencoglu et al. — "HARK Side of Deep Learning" (2019)
- **Link**: https://arxiv.org/abs/1904.07633
- Grad Student Descent: trial-and-error search with post-hoc explanation.
  HARKing: presenting post-hoc hypotheses as pre-registered.
- **Used in**: Decide (degeneration warning, named anti-patterns)

---

## Research Process

### Jia-Bin Huang — "Awesome Tips" (ongoing)
- **Link**: https://github.com/jbhuang0604/awesome-tips
- Work backward from the goal (assume perfect intermediates to find bottlenecks).
  Find proxy experiments with short turnaround for fast iteration.
- **Used in**: Plan (work backward, fast proxy)

### Shahid et al. — "Literature-Grounded Novelty Assessment" (2025)
- **Link**: https://arxiv.org/abs/2506.22026
- Single keyword searches retrieve ~3-4% of relevant work. Multi-stage
  retrieval with varied terminology achieves ~13% higher expert agreement.
- **Used in**: Triage (novelty scan with 3+ query variations), Principles (web search
  requirement for novelty/SOTA assessment)
