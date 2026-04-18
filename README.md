# Nexus Agent Benchmark — Eval Suite

A structured evaluation suite for measuring the Nexus Agent's Level 1 (Copilot) capability across real-world CAD and CAE engineering tasks.

## Current Count

| Category       | Count | Status |
|----------------|-------|--------|
| CAD evals      | 10    | Draft  |
| CAE evals      | 10    | Draft  |
| Infeasible     | 5     | Draft  |
| **Total**      | **25**| Draft  |

## Directory Structure

```
nexus-evals/
├── README.md                  # This file
├── DEVELOPER_GUIDE.md         # How to run evals, interpret scores, improve the agent
├── EXPERT_GUIDE.md            # How to create evals, validate, score runs, calibration protocol
├── EVAL_SPEC_TEMPLATE.yaml    # Blank template with field-level comments
├── cad/
│   └── CAD-{N}-{CATEGORY}-{TOOL}-{SEQ}/
│       ├── spec.yaml          # Full eval specification
│       ├── prompt.txt         # Exact instruction given to Nexus agent
│       ├── guide.md           # Tribal knowledge checklist (10-25 items)
│       ├── input/             # Input files (CAD models, STEP, etc.)
│       ├── expected/          # Reference screenshots, expected outputs
│       └── results/           # Populated after each eval run
├── cae/
│   └── CAE-{N}-{CATEGORY}-{TOOL}-{SEQ}/
│       └── (same structure)
└── infeasible/
    └── NEG-{TYPE}-{SEQ}/
        └── (same structure)
```

## How to Add an Eval

1. Copy `EVAL_SPEC_TEMPLATE.yaml` into a new eval directory as `spec.yaml`.
2. Fill in all fields. See existing evals for examples.
3. Write `prompt.txt` — a realistic, intentionally vague user prompt (1-4 sentences). Real users don't provide detailed specs; they describe what they need at a high level and expect the agent to make engineering assumptions.
4. Write `guide.md` — 10-25 actionable checklist items encoding expert tribal knowledge. The guide compensates for prompt ambiguity — it is NOT given to the user, it is embedded in the agent's harness.
5. Place input files in `input/`. Reference screenshots and expected outputs in `expected/`.
6. Get expert sign-off before changing `status` from `draft` to `validated`.

## How to Run an Eval

1. Load the input files into the target tool (SolidWorks, Abaqus, etc.).
2. Provide the agent with `prompt.txt` and `guide.md`.
3. Record the agent's full API call log, screenshots, and final model state.
4. Place all outputs in `results/run-YYYY-MM-DD/`.

## How to Score

See the "Evaluating Agent Runs" section in [EXPERT_GUIDE.md](EXPERT_GUIDE.md) for the full scoring protocol.

Summary:
- **Pass**: All success criteria met.
- **Partial**: >= 70% of criteria met, no critical failures.
- **Fail**: < 70% or any critical failure.
- **Infeasible evals**: Scored on detection, explanation quality, and remediation suggestions.

## Headline Metrics

- **Pass@1 rate**: Percentage of evals the agent passes on the first attempt with no human correction.
- **Latency**: Wall-clock time from prompt to agent signaling completion.

Track both weekly, broken down by category.

## Prompt Philosophy

Prompts are intentionally vague and ambiguous — they mirror how real engineers actually interact with the agent. A user does not say "remove all fillets under 2mm, suppress cosmetic features, retain thermal pads as separate bodies..." — they say "clean this up for thermal FEA." The agent must infer missing details, make engineering assumptions, and document what it assumed. This is a core capability being evaluated.
