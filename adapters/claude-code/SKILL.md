---
name: dual-paper-review-claude-code
version: v1.0
description: Dual-model academic paper proofreading using Claude Code subagents. Two subagents independently review a paper, then negotiate consensus through a parent-agent coordination layer until aligned scores and a unified revision plan are produced.
requires:
  - skill: dual-paper-review-spec
---

# Dual-Paper-Review — Claude Code Adapter

Place `paper_final.md` and `Detailed_Comments.md` in your working directory, then invoke `/skill dual-paper-review-claude-code`.

This adapter is read by the **parent agent** (Claude Code itself when you invoke the skill). The parent agent follows this specification to spawn two reviewer subagents, coordinate consensus rounds, and produce a final revision. The parent agent exercises judgment at every step — this document is the authoritative reference, not an executable script.

---

## Invocation Contract

When you invoke `/skill dual-paper-review-claude-code`, the following happens:

1. **You** (the user) place `paper_final.md` and `Detailed_Comments.md` in the working directory
2. **Parent agent** reads this SKILL.md and the shared `SPEC.md` for the 10-criterion rubric
3. **Parent agent** executes the 3-phase workflow described below, spawning subagents and evaluating their outputs
4. **Output** appears in `./D-Review/` when complete

**No additional setup required.** The parent agent handles all subagent spawning, result collection, and validation autonomously.

---

## Core Principle: The Parent Agent is Autonomous

This SKILL.md is a **specification document** that the parent agent reads and follows. The parent agent is a capable LLM that can:

- Spawn subagents using the `Agent` tool
- Collect and parse their outputs
- Execute the fence validation protocol (Layer 1, 2, 3) independently
- Compute score deltas and make alignment decisions
- Retry failed subagent calls
- Write files to `./D-Review/`
- Self-correct when outputs are malformed or incomplete

**Trust the parent agent to exercise judgment.** Write instructions as directives, not as executable pseudocode. The parent agent interprets and acts.

---

## Model Selection: Cheapest Available

### Step 0.1: Select the Most Cost-Effective Model

The parent agent queries available models in the current environment and selects the **cheapest available model** that meets minimum capability requirements.

**Selection criteria (in priority order):**
1. Must be able to follow the 10-criterion rubric with method branching
2. Must be able to produce structured JSON scorecards
3. Prefer models with lower cost per token

**Do NOT hardcode model names.** The parent agent detects what is available in the current Claude Code environment and selects appropriately.

### Step 0.2: Log Initialization

Write a single trace entry to `trace.jsonl`:
```
{"trace_id": "init-001", "step": "model_initialization", "model_selected": "<model>", "timestamp": "<ISO-8601>"}
```

---

## Input Variables

Before starting, the parent agent determines:

| Variable | How Determined |
|----------|----------------|
| `{{PAPER_CONTENT}}` | Read from `paper_final.md` in working directory |
| `{{PEER_REVIEW_COMMENTS}}` | Read from `Detailed_Comments.md` in working directory |
| `{{METHODOLOGY}}` | Auto-detect: scan paper for `qualitative`/`quantitative`/`mixed` keywords. Default to `mixed` if ambiguous. |

---

## Phase 1: Independent Parallel Review

### 1.1: Hash and Protect Source Files

Before any processing:
1. Compute SHA256 of `paper_final.md` → store in `D-Review/hashes.json`
2. Compute SHA256 of `Detailed_Comments.md` → store in `D-Review/hashes.json`
3. Copy both files to `D-Review/input/` (read-only)
4. Create `./D-Review/independent/` and `./D-Review/rounds/` directories

### 1.2: Spawn Reviewer A and Reviewer B in Parallel

**Spawn two subagents concurrently** using the `Agent` tool with `run_in_background: true`:

**Subagent A** receives:
```
Read references/reviewer-1-prompt.md and substitute:
  {{PAPER_CONTENT}} → <full paper text>
  {{PEER_REVIEW_COMMENTS}} → <full comments text>
  {{METHODOLOGY}} → <detected methodology>
Then produce a complete independent review following the prompt template exactly.
```

**Subagent B** receives:
```
Read references/reviewer-2-prompt.md and substitute:
  {{PAPER_CONTENT}} → <full paper text>
  {{PEER_REVIEW_COMMENTS}} → <full comments text>
  {{METHODOLOGY}} → <detected methodology>
Then produce a complete independent review following the prompt template exactly.
```

**Await both results using `TaskOutput`** until both subagents return. Do not proceed until both are complete.

### 1.3: Parse and Validate Outputs

For each subagent output:

**Layer 1 (Structural)**: Is the JSON scorecard parseable? Are all 10 criteria present with integer scores [1-5]? Does overall_score match the mean of 10 criteria (±0.1)?

**Layer 2 (Evidence)**: Do the evidence quotes exist verbatim in the source paper? Strip any that cannot be verified.

**Layer 3 (Consistency)**: Does the score narrative match the numeric rating? Does the change summary reflect actual annotations?

### 1.4: Handle Validation Failures

| Condition | Action |
|-----------|--------|
| Layer 1 fails | Respawn the same reviewer once. If still failing, log error and continue with best available output. |
| Layer 2 fails (< 3 hallucinated) | Strip bad quotes, flag affected criteria, continue |
| Layer 2 fails (≥ 3 hallucinated) | Respawn once. If still failing, log error and continue. |
| Layer 3 fails | Log warning, continue |

### 1.5: Save Independent Review Outputs

Write to `D-Review/independent/`:
- `reviewer-a-output.md` — Full markdown output from Subagent A
- `reviewer-a-scorecard.json` — Extracted JSON scorecard
- `reviewer-b-output.md` — Full markdown output from Subagent B
- `reviewer-b-scorecard.json` — Extracted JSON scorecard

Write `D-Review/fence-results/reviewer-a-fence.json` and `reviewer-b-fence.json` with validation results.

Write trace entry:
```
{"trace_id": "trace-002", "phase": "independent_review", "step": "phase1_complete", "reviewer_a_score": <>, "reviewer_b_score": <>}
```

---

## Phase 2: Iterative Consensus Negotiation

### 2.1: Initialize Loop State

```
round = 1
best_round = null
best_overall = 0
stagnation_count = 0
```

### 2.2: Spawn Consensus Subagents (Parallel)

**For Round 1**: Feed both independent review scorecards and revisions to **both** consensus subagents.

**For Round > 1**: Feed the prior round's consensus outputs + score deltas + fence results to **both** consensus subagents.

Each consensus subagent receives:
```
Read references/consensus-prompt.md and substitute all placeholders:
  {{PAPER_CONTENT}}, {{PEER_REVIEW_COMMENTS}}, {{METHODOLOGY}}
  {{REVIEWER_1_SCORECARD}}, {{REVIEWER_2_SCORECARD}}
  {{REVIEWER_1_REVISION}}, {{REVIEWER_2_REVISION}}
  {{ROUND_NUMBER}}, {{PREVIOUS_CONSENSUS}}
Then produce a consensus revision following the prompt template exactly.
```

**Spawn both in parallel with `run_in_background: true`**, await both with `TaskOutput`.

### 2.3: Parent Agent Computes Score Deltas

After both consensus outputs return, the parent agent parses their JSON scorecards and computes:
```
For each criterion i: delta_i = |reviewer_a_score[i] - reviewer_b_score[i]|
overall_delta = |reviewer_a_overall - reviewer_b_overall|
```

### 2.4: Alignment Check

Consensus is achieved when ALL of the following are true:
- At least 8 of 10 criterion deltas == 0; remaining deltas <= 1 (no delta > 1)
- Both overall scores >= 4.3
- No criterion score < 4 in either model
- Top-5 priority changes overlap >= 3 of 5

If ALL pass → **CONSENSUS ACHIEVED** → proceed to Phase 3.

### 2.5: Revision Plan Convergence Check

Extract top-5 priority changes from each consensus output. If overlap >= 3 and score alignment holds → consensus achieved.

### 2.6: Improvement Check

```
if max(reviewer_a_overall, reviewer_b_overall) > best_overall + 0.1:
  best_round = round
  best_overall = max(reviewer_a_overall, reviewer_b_overall)
  stagnation_count = 0
else:
  stagnation_count += 1
```

### 2.7: Stagnation Check

If `stagnation_count >= 3` → STOP → carry best-so-far to Phase 3.

### 2.8: Save Round Artifacts

Write to `D-Review/rounds/round-{NNN}/`:
- `reviewer-a-consensus.md`, `reviewer-a-scorecard.json`
- `reviewer-b-consensus.md`, `reviewer-b-scorecard.json`
- `score-delta.json`
- `round-verdict.json` (`{"status": "aligned" | "diverged", "best_model": "a"|"b"|null}`)
- `round-checkpoint.json`

Write trace entry with round number, scores, deltas, fence results.

### 2.9: Loop or Exit

- If consensus achieved → proceed to Phase 3
- If stagnation_count >= 3 → proceed to Phase 3 with best-so-far
- If round >= 10 → proceed to Phase 3 with best-so-far
- Otherwise → round += 1, go to Step 2.2

---

## Phase 3: Final Output & Validation

### 3.1: Select Final Artifact

- If consensus was reached in an earlier round → use that round's aligned output
- Otherwise → use best-so-far from the round with highest `best_overall`

### 3.2: Final Comprehensive Validation

Re-verify:
- All evidence quotes in final output against source text in `D-Review/input/`
- Every item in `Detailed_Comments.md` is addressed by the revision plan
- Revision plan is specific and actionable (reject vague directives)
- No hallucinated content
- Source file hashes still match `D-Review/hashes.json` (CRITICAL CHECK)
- `validation-report.json` must include a `source_hash_verified` boolean field set to `true` only if hashes match. If false, the entire validation-report.json must set `validation_passed: false` and include `"hash_mismatch": true`.

### 3.3: Write Final Deliverables

Write to `D-Review/final/`:
- `consensus-revision.md` — Final revised manuscript
- `consensus-scorecard.json` — Final agreed scorecard
- `revision-plan.md` — Numbered actionable directives
- `change-summary.md` — Consolidated change summary
- `negotiation-log.md` — Summary of all rounds
- `validation-report.json` — Pass/fail per validation check. **MUST include:**
  - `"source_hash_verified": true|false` — true only if all source hashes match
  - `"validation_passed": true|false` — false if any critical check fails including hash mismatch
  - `"hash_mismatch": true|false` — set to true if source hash does not match stored hash
  - If `hash_mismatch` is true, `validation_passed` MUST be `false`
- `status.json` — `{"status": "consensus"|"best-effort"|"hash_mismatch_error", "rounds_used": N, "final_overall_score": "X.X", "source_hash_verified": true|false}`. **If source hash mismatch is detected, `status` MUST be `"hash_mismatch_error"` — never `"consensus"` or `"best-effort"`**.

### 3.4: Final Source Hash Check

Compute SHA256 of original `paper_final.md` and `Detailed_Comments.md`. Compare against `D-Review/hashes.json`. If mismatch → write `D-Review/errors/error-report.md` and **HALT** without producing final output.

---

## Self-Correction Guidance for the Parent Agent

The parent agent should self-correct in these situations:

| Situation | Self-Correction Action |
|----------|----------------------|
| Subagent output is missing the JSON scorecard block | Extract scorecard from markdown if embedded; otherwise respawn |
| Subagent output is missing required sections (SCORECARD/ANNOTATED REVISION/CHANGE SUMMARY) | Respawn with stricter instructions about required sections |
| Evidence quotes are not verifiable | Strip unverifiable quotes, flag criterion, continue |
| Score delta computation reveals disagreement | Proceed to next consensus round with both outputs as input |
| Subagent returns malformed JSON | Attempt to parse with loose JSON parsing; if fails, respawn |
| Round is taking too long (>5 rounds with no improvement) | Check stagnation_count; if >= 3, terminate loop |
| Fence validation repeatedly fails for same subagent | Try the other model/approach; log error, continue with best available |

**The parent agent has full authority to interpret, retry, and adapt within the bounds of this specification.**

---

## Key Differences from oh-my-pi Adapter

| Aspect | oh-my-pi | Claude Code Adapter |
|--------|----------|-------------------|
| **Model routing** | Different models via role aliases | Same model for both reviewers (parent selects cheapest available) |
| **Parallelism** | True parallel inference (different models) | Concurrent subagent spawning (same model, parallel) |
| **State** | Shared filesystem artifact | Parent agent holds all state between spawns |
| **Fallback** | Model-level fallback chain | Retry on same model (primary mechanism) |

---

## Shared Reference Files

This adapter uses these shared files (relative to project root):
- `SPEC.md` — Shared core specification (read by parent agent for rubric)
- `references/reviewer-1-prompt.md` — Reviewer A prompt template
- `references/reviewer-2-prompt.md` — Reviewer B prompt template
- `references/consensus-prompt.md` — Consensus prompt template

---

## Complete File Structure Created

```
./D-Review/
├── input/
│   ├── paper_final.md
│   └── Detailed_Comments.md
├── independent/
│   ├── reviewer-a-output.md
│   ├── reviewer-a-scorecard.json
│   ├── reviewer-b-output.md
│   └── reviewer-b-scorecard.json
├── fence-results/
│   ├── reviewer-a-fence.json
│   ├── reviewer-b-fence.json
│   └── round-*-reviewer-*.json
├── rounds/
│   └── round-001/ ... round-010/
│       ├── reviewer-a-consensus.md
│       ├── reviewer-b-consensus.md
│       ├── score-delta.json
│       ├── round-verdict.json
│       └── round-checkpoint.json
├── final/
│   ├── consensus-revision.md
│   ├── consensus-scorecard.json
│   ├── revision-plan.md
│   ├── change-summary.md
│   ├── negotiation-log.md
│   ├── validation-report.json
│   └── status.json
├── errors/
│   └── error-report.md
├── trace.jsonl
└── hashes.json
```

---

## Termination Conditions Summary

| Condition | Action |
|-----------|--------|
| All alignment criteria met | CONSENSUS — use aligned output |
| stagnation_count >= 3 | STOP — best-so-far |
| round > 10 | STOP — best-so-far |
| >= 3 hallucinated criteria after retry | STOP — error report |
| Source hash mismatch | CRITICAL STOP — error report |
