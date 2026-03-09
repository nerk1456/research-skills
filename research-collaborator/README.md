# research-collaborator

A Claude Code skill that guardrails your research workflow. It encodes the principles that experienced researchers follow and applies them before you spend the GPU hours.

## Why

Good research advice exists: Karpathy wrote down how to train neural networks. Lipton & Steinhardt catalogued how ML papers fool themselves. Kapoor & Narayanan mapped out every way data leaks. But it's hard to remember all that at 2AM when you are convinced your idea just needs one more run. This skill puts that knowledge in the loop: checks your hypothesis before you commit, catches known bugs before you blame the idea, and calls out sloppy methodology before a reviewer does.

## Modes

| Mode | You say | What happens |
|------|---------|---------|
| **TRIAGE** | "I have an idea" | Turns it into a falsifiable hypothesis, searches prior work (3+ query variations), recommends GO / REVISE / KILL |
| **PLAN** | "Design experiments for this" | Breaks hypothesis into testable sub-claims, specifies baselines, ablations, and kill criteria |
| **EXECUTE** | "Let's test it" | Reads your codebase, implements changes, runs sanity checks before the real experiment |
| **DIAGNOSE** | "Why isn't this working?" | Reads code and logs, separates bugs from bad ideas |
| **VALIDATE** | "I got results" | Checks for data leakage, HP attribution confounds, seed sensitivity, anticipates reviewer objections |

## Features

- Kill criteria on every hypothesis, designed to disprove, not confirm
- Reads your code and logs instead of asking you to describe symptoms
- Flags overclaiming, HARKing, and Grad Student Descent before you write the paper
- 191 silent bugs (no error, wrong results) organized by architecture: transformers, diffusion, VQ-VAE, GANs, RL, GNNs, detection, segmentation, seq2seq, contrastive learning, NeRF/3DGS, flow matching

## Usage

```
/research-collaborator
```

Or just talk:

```
"I have an idea for using score-guided diffusion to steer motion generation"
"My transformer's attention maps look uniform after layer 4"
"Training loss plateaus at 0.3 and won't go lower"
"Are these results solid or am I missing something?"
```

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Modes, protocols, methodology (533 lines) |
| `silent-bugs.md` | 191 silent bugs, 16 tiers, 13+ architectures (2400 lines) |
| `sources.md` | Bibliography |
