# Dual-Paper-Review

**AI-powered triangulation review for academic papers.** A high-capability model generates expert peer-review comments; two cheap models independently review the manuscript and negotiate consensus through iterative rounds until aligned scores and a unified revision plan emerge.

---

## Architecture

```
┌───────────────────────────────────────────────────────────────-──┐
│  Step 1: EXPERT REVIEW GENERATION (High-End Model)               │
│                                                                  │
│  Upload paper_final.md to Gemini 3.1 Pro / GPT 5.x / Claude      │
│  → Receive Detailed_Comments.md (structured peer-review feedback)│
└────────────────────────┬────────────────────────────────────────-┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 2: DUAL-MODEL CONSENSUS REVIEW (Cheap Models)             │
│                                                                 │
│  ┌──────────────┐          ┌──────────────┐                     │
│  │ Reviewer A   │          │ Reviewer B   │                     │
│  │ (Academic    │◄────────►│ (Structure & │                     │
│  │  Depth)      │  Score   │  Clarity)    │                     │
│  └──────┬───────┘  Delta   └──────┬───────┘                     │
│         │    ◄──────────────────► │                             │
│         │         Round N         │                             │
│         └───────────┬─────────────┘                             │
│                     ▼                                           │
│         ┌───────────────────────┐                               │
│         │ Consensus Scorecard + │                               │
│         │ Annotated Revision    │                               │
│         │ Revision Plan         │                               │
│         └───────────────────────┘                               │
│                     │                                           │
│         ◄─────────-─┴──────────►                                │
│              Up to 10 Rounds                                    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step 3: FINAL OUTPUT (Publication-Ready)                       │
│                                                                 │
│  consensus-revision.md     ← Bold/strikethrough annotations     │
│  consensus-scorecard.json  ← Machine-parseable scores           │
│  revision-plan.md         ← Numbered actionable directives      │
│  negotiation-log.md       ← Full round-by-round trace           │
└─────────────────────────────────────────────────────────────────┘
```

---

## Why This Exists

### The Inspiration

This project is inspired by [Andrej Karpathy's AutoResearch](https://github.com/karpathy/autoresearch), which demonstrated that autonomous iteration through structured artifacts can produce self-improving AI systems. AutoResearch's core insight — that an AI can critique its own outputs, negotiate with itself across rounds, and converge toward better solutions through explicit acceptance criteria — is the backbone of this tool.

### The Problem with Traditional AI Review

Conventional AI-assisted paper review requires **lengthy, iterative conversations** with a chatbot: paste the paper, receive comments, ask for clarifications, request revisions, verify changes, repeat. This is:

- **Token-intensive**: Each round costs API credits
- **Time-consuming**: Human must guide the entire dialogue
- **Inconsistent**: Different prompts yield wildly different feedback quality
- **Non-reproducible**: No audit trail of what was changed and why

### The Solution

Dual-Paper-Review streamlines this into **one high-end model interaction** for expert review generation, followed by fully automated multi-round consensus negotiation among cheap models. You get:

- **Significant cost savings**: High-end model used only once for review generation
- **Consistent methodology**: 10-criterion rubric applied uniformly across rounds
- **Full transparency**: Every negotiation round logged and traceable
- **Less human effort**: No need to guide tedious back-and-forth dialogue

### Why a Revision Plan, Not Direct Edits?

We deliberately output a **revision plan** rather than a fully revised manuscript. This is a design choice grounded in how AI-assisted academic work should work:

1. **AI hallucinations are unavoidable**: Even the most capable models occasionally produce confident nonsense. In academic writing, undetected errors can be catastrophic — a plausible but wrong statistical interpretation or a fabricated citation could doom a paper's credibility.

2. **You are the principal investigator**: The paper is your intellectual contribution. The AI's role is to identify issues and suggest directions; you decide what to implement. This keeps **you in the principal position** — reviewing, accepting, modifying, or rejecting each suggestion based on your expert judgment.

3. **Transparency over black-box magic**: A revision plan makes the reasoning explicit. You see *why* a change is suggested, what evidence anchors it, and how urgent it is (via quantified scores). This is far more useful than receiving a "revised" document with no explanation.

4. **Ownership and integrity**: When your name is on the paper, you own every word. Accepting AI-generated edits without understanding them creates risk. The revision plan workflow ensures you engage meaningfully with every change.

---

## Key Features

| Feature | Description |
|---------|-------------|
| **Triangulation** | Two reviewers with distinct focuses negotiate consensus, reducing single-model bias |
| **Quantified Scores** | 10-criterion rubric (1–5 per criterion) with evidence anchors |
| **Methodology-Aware** | Criteria 5–6 branch based on qualitative, quantitative, or mixed-method research |
| **Self-Validating** | 3-layer fence protocol verifies evidence quotes against source text |
| **Fully Traceable** | Every round logged; full negotiation history preserved |
| **Source Protection** | Original files never modified; SHA256 hash verification before/after |

---

## Two Adapters, One Skill

Choose the adapter that matches your execution environment:

| Adapter | Best For | Model Routing |
|---------|----------|--------------|
| **oh-my-pi** | Dual-model setups (GLM + MiniMax via role switching) | Different models per role |
| **Claude Code** | Claude Code environments with subagent spawning | Same model for both reviewers |

---

## Quick Start

### Step 1: Generate Expert Review Comments

Upload your paper to a high-capability model:

```
Upload paper_final.md and prompt:
"Review this academic paper and generate detailed peer-review comments
in markdown format. Key each comment to specific sections/paragraphs.
Output as Detailed_Comments.md."

Save the response as Detailed_Comments.md in your working directory.
```

### Step 2: Run Dual Review

**For Claude Code:**
```
/skill dual-paper-review-claude-code
```

**For oh-my-pi:**
```
/skill dual-paper-review-oh-my-pi
```

### Step 3: Review Output

Results appear in `./D-Review/`:
```
./D-Review/
├── final/
│   ├── consensus-revision.md      # Annotated paper with changes
│   ├── consensus-scorecard.json   # Final scores (e.g., 4.3/5.0)
│   ├── revision-plan.md           # Actionable revision list
│   └── negotiation-log.md         # How consensus was reached
└── rounds/                       # All rounds preserved
```

---

## The 10-Criterion Rubric

All reviewers score across 10 criteria. Criteria 5 and 6 branch based on methodology.

| # | Criterion | Applies To |
|---|-----------|-----------|
| 1 | Structural Coherence & Logical Flow | All |
| 2 | Completeness & Section Coverage | All |
| 3 | Clarity & Precision of Language | All |
| 4 | Theoretical Framework Consistency | All |
| 5 | Methodological Appropriateness | **Branches** |
| 6 | Results & Interpretation | **Branches** |
| 7 | Discussion & Implications | All |
| 8 | Integration of Peer Review Feedback | All |
| 9 | Language Mechanics & Grammar | All |
| 10 | Formatting & Consistency | All |

### Criterion 5 & 6 — Methodology Branching

| Criterion | Qualitative | Quantitative | Mixed |
|-----------|-------------|-------------|-------|
| **Methodological Appropriateness** | Trustworthiness strategies (credibility, transferability, dependability, confirmability) | Validity threats, statistical assumptions, reliability metrics | Both strands + integration rationale |
| **Results & Interpretation** | Thematic depth, quote integration | Effect sizes, confidence intervals, p-values | Strand integration, meta-inferences |

---

## Consensus Conditions

Consensus is achieved when ALL hold:

- At least 8 of 10 criterion deltas == 0, remaining deltas <= 1 (no delta > 1)
- Both overall scores >= 4.3
- No criterion below 4 in either model
- Top-5 priority changes overlap >= 3 of 5

If not reached after 10 rounds → best-so-far output is used.

---

## Source File Protection

- Original `paper_final.md` and `Detailed_Comments.md` are **never modified**
- SHA256 hashes computed at start and end — workflow halts if hashes change
- All processing isolated in `./D-Review/` directory
- Copy (not symlink) of source files used for all operations

---

## Requirements

| Item | Requirement |
|------|-------------|
| Paper format | Markdown (.md), IMRaD or variant structure |
| Comments format | Markdown (.md), structured feedback |
| Output boundary | All outputs in `./D-Review/` only |
| Review language | Always English |
| Invocation | Single paper per run |

---

## File Structure

```
Dual-Paper-Review/
├── SPEC.md                          # Shared core specification
├── README.md                         # This file
├── references/                       # Prompt templates (shared)
│   ├── reviewer-1-prompt.md
│   ├── reviewer-2-prompt.md
│   └── consensus-prompt.md
└── adapters/                        # Platform-specific
    ├── oh-my-pi/
    │   └── SKILL.md
    └── claude-code/
        └── SKILL.md
```

---

## License

**[CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/)** — Creative Commons Attribution-NonCommercial 4.0 International

This license was chosen intentionally for an academic tool:
- **Non-commercial**: Prohibits use in commercial products or paid services
- **Attribution required**: Credit must be given to contributors
- **Share-alike**: Derivative works must be distributed under the same license
- Researchers and institutions are free to use and modify this tool

---

## Citation

If Dual-Paper-Review contributed to your published work, cite it as:

```bibtex
@software{dual_paper_review,
  author    = {Xuao Li},
  title     = {Dual-Paper-Review: AI-Powered Triangulation Review for Academic Papers},
  year      = {2026},
  url       = {https://github.com/StP00h/Dual-Paper-Review},
  version   = {1.0},
  license   = {CC BY-NC 4.0}
}
```
