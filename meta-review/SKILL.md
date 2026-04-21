---
name: meta-review
description: Simulate 4 independent reviewer perspectives, perform self-verification, identify consensus vs individual concerns. Synthesize into a meta-review that separates signal from noise.
---

# Meta-Review

Simulate multiple independent reviewer perspectives and synthesize.

## Step 1: Multi-Perspective Simulation

Simulate 4 independent reviewers with different focus areas:
- **Reviewer A:** Methodology and technical correctness
- **Reviewer B:** Literature context and novelty
- **Reviewer C:** Presentation quality and clarity
- **Reviewer D:** Practical relevance and impact

Each reviewer evaluates independently. For each, note:
- Top-3 strengths
- Top-3 weaknesses
- Confidence (1-10)

## Step 2: Self-Verification

Re-read your own findings. For each:
- Is it supported by the text? (cite the passage)
- Could another reviewer disagree?
- Adjust confidence if needed

## Step 3: Consensus Analysis

| Concern | Rev.A | Rev.B | Rev.C | Rev.D | Type |
|---------|-------|-------|-------|-------|------|
| ...     | Y     | Y     | Y     |       | CONSENSUS-3 |
| ...     | Y     |       |       |       | INDIVIDUAL |

- **CONSENSUS-4**: All agree → MUST be addressed
- **CONSENSUS-3**: 3 of 4 → SHOULD be addressed
- **INDIVIDUAL**: Only one reviewer → CAN be addressed

## Step 4: Write Meta-Review

Produce:
1. Consensus concerns (ranked by agreement)
2. Individual concerns
3. Agreed-upon strengths
4. Overall assessment (synthesis of all perspectives)

For each consensus concern, create a separate finding.
