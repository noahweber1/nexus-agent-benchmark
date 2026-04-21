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

### Folder Roles: input/ vs expected/ vs results/

`input/`, `expected/`, and `results/` all hold the same kinds of files (e.g., all three can contain a `.sldprt`), but they play different roles:

| Folder | What it contains | When written | Multiplicity |
|---|---|---|---|
| `input/` | Starting state — the **native program file** (e.g., `.sldprt`, `.cae`, `.mph`) the agent opens along with `prompt.txt` | When the eval is authored | One immutable set |
| `expected/` | **Runnable reproduction bundle**: native program file + text solver deck or journal (required where the tool has one) + reference results + `environment.yaml` + `fingerprint.yaml` + `DECISIONS.md` + `REPRODUCE.md` + 3–5 screenshots. The native file is produced by the expert performing the task by hand in the target software, as they would without any agent. See [EXPERT_GUIDE.md](EXPERT_GUIDE.md) §End-File Package for the per-tool artifact list and pre-handoff anonymization checklist | When the eval is authored | One immutable set |
| `results/run-YYYY-MM-DD/` | A single run's artifacts: the agent's **native program file** plus screenshots, logs, and evaluator score | Every eval run | Many, grows over time |

#### The end state must be the native program file, not a neutral export

The target we are aiming for is the native output of the CAD/CAE program we are operating — e.g., a SolidWorks `.sldprt`, an Abaqus `.cae`, a COMSOL `.mph`, an ANSYS `.wbpj`. These formats preserve the **parametric feature tree**: fillets, extrudes, patterns, mates, BCs, material assignments, mesh controls, study definitions. The evaluator opens the file in the target tool and inspects the modifications parametrically — clicking into the feature tree, editing dimensions, re-running the solve.

Neutral exports (`.step`, `.iges`, `.stl`, `.pdf`, `.png`) are **dead artifacts**: they capture geometry or a view but lose the feature tree, constraints, and design intent. They are useful as *supplementary evidence* for visual comparison and cross-tool review, but they are never the reference answer. If a `.step` file is the only thing in `expected/`, the eval is broken — there is no way to verify that the agent modified the model tree correctly.

Flow: `input/` + `prompt.txt` → agent → `results/run-*/output_files/` (native file) → evaluator opens native file in the target tool and compares against `expected/` using `success_criteria`. See [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md) for the complete `results/` layout and per-tool file-extension reference.

### Reproduction bar

The `expected/` folder is not just a reference — it is a **runnable bundle**. The contract: another engineer who has the target tool installed must be able to open the folder and reproduce the end state in under 30 minutes of wall-clock work, using only what is in the folder. If they have to email the author to ask how it was built, the bundle is incomplete.

See [EXPERT_GUIDE.md](EXPERT_GUIDE.md) §End-File Package for the per-tool artifact matrix (SolidWorks, Workbench, Mechanical, Fluent, LS-DYNA, Abaqus, COMSOL) and the universal tail (`tool_version.txt`, `REPRODUCE.md`, neutral geometry export, screenshots).

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
