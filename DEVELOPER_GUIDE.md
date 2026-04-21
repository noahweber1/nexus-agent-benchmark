# Developer Guide — Nexus Agent Benchmark

How to run evals, interpret results, incorporate expert feedback, and use findings to improve the agent.

## Your Role

You are a developer working on the Nexus Agent. Your job is to:

1. **Run evals** against the current agent build
2. **Record results** in a structured format
3. **Interpret expert scores** and translate them into agent improvements
4. **Update guides** (the `.md` files that feed into the agent's harness)
5. **Track progress** across eval runs over time

You are NOT expected to judge engineering correctness — that is the expert's job. You are expected to understand the eval infrastructure, execute runs cleanly, and translate expert feedback into actionable engineering tasks.

---

## Understanding the Eval Structure

Each eval directory contains:

| File | What it is | Who writes it | Who reads it |
|------|-----------|---------------|-------------|
| `spec.yaml` | Machine-readable eval definition | Expert + Developer | Both |
| `prompt.txt` | The vague, realistic user prompt given to the agent | Expert (validated) | Agent |
| `guide.md` | Tribal knowledge checklist — embedded in agent harness | Expert | Agent (via harness) |
| `input/` | Starting files (CAD models, STEP, meshes) | Expert provides or Developer reconstructs | Agent |
| `expected/` | Reference end file(s) produced by the expert — the "correct answer" | Expert provides | Evaluator |
| `results/` | Agent outputs + evaluator scores (per run) | Developer records, Expert scores | Both |

### Folder Roles: input/ vs expected/ vs results/

`input/`, `expected/`, and `results/` all hold the same *kinds* of files (e.g., all three can contain a `.sldprt`), but they play different roles in the eval lifecycle:

| Folder | What it contains | When written | Multiplicity |
|---|---|---|---|
| `input/` | Starting state — the **native program file** (e.g., `.sldprt`, `.cae`, `.mph`) the agent opens along with `prompt.txt` | Once, when the eval is authored | One immutable set |
| `expected/` | The expert's reference answer — the **native program file** with both model tree and geometry modified to the correct final state | Once, when the eval is authored | One immutable set |
| `results/run-YYYY-MM-DD/output_files/` | The agent's actual attempt — the **native program file** the agent produced, plus derived artifacts | Every time the eval is executed | Many, one per run |

**The end state target is the native program file, not a neutral export.** Native formats (`.sldprt`, `.cae`, `.mph`, `.wbpj`, etc.) preserve the parametric feature tree — the evaluator opens the file in the target tool and inspects fillets, extrudes, patterns, mates, BCs, and study definitions directly. Neutral exports (`.step`, `.iges`, `.stl`) are dead geometry: useful as supplementary evidence for visual comparison, but useless as the reference answer because the model tree is gone.

The flow: `input/` + `prompt.txt` → agent → `results/run-*/output_files/` (native file) → evaluator opens native file in the target tool and compares against `expected/` using `success_criteria` in `spec.yaml`.

`expected/` is the authoritative reference and never changes once signed off. `results/` accumulates over time and captures how the agent behaves across builds.

### Reading spec.yaml

Key fields to understand:

```yaml
complexity:
  geometric: 4          # How complex is the geometry? (1=prismatic, 5=freeform)
  engineering_judgment: 5 # How much expert decision-making? (1=cookbook, 5=judgment-heavy)
  tool_proficiency: 4    # How much tool-specific knowledge? (1=basic, 5=power-user)
  estimated_tool_calls: 55 # Expected number of API/tool calls to complete

success_criteria:        # THE SCORECARD — this is what the expert evaluates against
  - id: "SC-01"
    description: "..."
    critical: true       # If critical=true and this FAILS, the whole eval FAILS
    type: "geometry_check"

infeasibility_flag: false # If true, the CORRECT behavior is to REFUSE the task
```

**Critical failures override everything.** Even if 9/10 criteria pass, one critical failure = overall FAIL. This is intentional — in engineering, certain errors are unacceptable regardless of what else went right.

### Reading prompt.txt

Prompts are intentionally vague — 1-4 sentences, casual tone, like a real engineer would type. This is by design.

**Do not "fix" vague prompts.** The agent's ability to handle ambiguity and make reasonable assumptions is a core capability being tested. If you think a prompt is too vague, discuss with the expert who wrote it before changing anything.

The prompt does NOT contain the guide. The guide is injected separately via the agent harness.

### Reading guide.md

The guide is the most product-relevant artifact in each eval. It encodes tribal knowledge — the 10-25 things an expert would tell a new hire before letting them attempt this task.

**These guides become agent Skills/Guides in the product.** When you improve a guide based on eval results, you are directly improving the product.

---

## Verifying the Reference Bundle

Before you run an agent against an eval, verify that the expert's `expected/` bundle itself reproduces. This is a cheap sanity check that catches the common failure modes — missing artifact, silent version upgrade, machine-state leakage, broken text deck — before you waste a full agent run comparing against a broken reference.

### What the bundle should contain

Per EXPERT_GUIDE.md §End-File Package:

- Runnable native file (`.wbpz` / `.cae` / `.sldprt` / `.mph` / …)
- Text solver deck or journal where the tool has one (`.inp`, `.k`, `.jou`, `.dat`, `.wbjn`)
- Reference results (`.rst`, `.odb`, `.dat.h5`, `d3plot`, or embedded in the native file)
- `environment.yaml`, `DECISIONS.md`, `REPRODUCE.md`
- 3–5 reference screenshots

If any of these is missing, refuse the eval back to the expert. Do not try to reconstruct what is missing — the whole point of the bundle is that you are not guessing.

### How to verify

The developer is expected to have the target tool installed (SolidWorks, Workbench, Abaqus, LS-DYNA, Fluent, COMSOL, etc.). Two verification modes, pick whichever is faster for the tool at hand:

**Batch replay (preferred where a text deck exists).** For Abaqus, LS-DYNA, Fluent, Mechanical APDL, Workbench: run the text deck headlessly, then extract the numerical targets named in `spec.yaml.success_criteria` from the solver output.

```bash
# Examples
abaqus job=input cpus=4 interactive
lsdyna i=input.k ncpu=4
fluent 3ddp -g -t4 -i solve.jou
mapdl -b -np 4 -i input.dat
Workbench.exe -B -R session.wbjn
```

Then extract the scalars named in `success_criteria` from the solver output (element counts from the `.odb`/`.rst`/`.dat.h5`, peak-stress via tool-specific post-processing, displacement at a named probe node) and assert each against its tolerance. Faster than GUI verification and scriptable.

**GUI verification.** For SolidWorks, Discovery, COMSOL, and Abaqus/CAE pre/post workflows: open the native file, follow `REPRODUCE.md`, extract the `success_criteria` scalars in the tool's post-processor, and check against their tolerances. Use the screenshots as cross-checks.

### Diagnosing mismatches

When a reference scalar fails to reproduce, the failure is usually one of:

- **Tool version drift** — cross-check `environment.yaml` vs. your install. Pin the tool version if drift is the cause.
- **Solver nondeterminism** — parallel reduction order, thread scheduling. Match `environment.yaml.cores_used` and rerun.
- **Machine state leakage** — absolute paths, missing custom material DB, unlisted addon. Check against the anonymization checklist in EXPERT_GUIDE.md.
- **Real bundle corruption** — the bundle is genuinely broken. Bounce it back to the expert.

### When to demand more from the expert

- `success_criteria` contains only subjective checks ("looks reasonable") — push back. Every numerical criterion needs a target value and a tolerance.
- `DECISIONS.md` is empty or reads like `REPRODUCE.md` — push back. It should record judgment calls, not steps.
- No text deck for a tool that supports one — push back. Abaqus/LS-DYNA/Fluent/APDL/Workbench evals without the text deck make regression runs expensive.

---

## Running an Eval

### Prerequisites

1. Target tool installed and licensed (SolidWorks, Abaqus, Ansys, etc.)
2. Input files placed in the eval's `input/` directory
3. Agent build deployed and accessible
4. Recording infrastructure ready (API call logging, screenshot capture)

### Execution Steps

1. **Create a run directory:**
   ```
   results/run-YYYY-MM-DD/
   ```

2. **Load input state.** Open the target tool, load the input files. Verify the starting state matches `spec.yaml > input_state > state_description`.

3. **Inject the prompt and guide.** Provide `prompt.txt` as the user message. Inject `guide.md` via the agent harness (not as user-visible text — the guide simulates embedded product knowledge, not user instructions).

4. **Record everything:**
   - Full API/tool call log (every call, in order)
   - Screenshots at key milestones and at completion
   - Final model tree state (screenshot or programmatic export)
   - Wall-clock time from prompt to completion

5. **Package the output.** Save all agent-produced files into `results/run-YYYY-MM-DD/`.

6. **Hand off to expert evaluator.** Provide them with the evaluator packet (see EXPERT_GUIDE.md).

### What to Record in results/

```
results/run-YYYY-MM-DD/
  output_files/          # Whatever the agent produced (models, drawings, reports)
  screenshots/           # Periodic + final screenshots
  model_tree.png         # Final model tree capture
  api_call_log.json      # Full tool call sequence
  score.yaml             # Filled in by the expert evaluator
  evaluator_notes.md     # Free-text expert observations
```

#### File Types to Produce in `output_files/`

The agent's native project file is the authoritative output — `expected/` contains the same kind of file, and the evaluator compares them directly. Derived artifacts (STEP, PDF, screenshots) are for easier visual / cross-tool comparison and review.

| Eval(s) | Target tool | Primary output extension(s) |
|---|---|---|
| CAD-1, 2, 3, 6, 7, 8, 9, 10 | SolidWorks | `.sldprt` (part), `.sldasm` (assembly), `.slddrw` (drawing) |
| CAD-4 | Creo | `.prt` (part), `.asm` (assembly), `.drw` (drawing) |
| CAD-5 | NX | `.prt` (Siemens NX — same extension as Creo, different format) |
| CAE-1 | ANSA | `.ansa`; solver decks `.nas`/`.bdf`/`.inp`/`.key` |
| CAE-2, 3, 10 | Abaqus | `.cae` (model DB), `.inp` (solver input), `.odb` (results), `.dat`/`.msg`/`.sta` (logs) |
| CAE-4, 5, 8 | ANSYS | `.wbpj`/`.wbpz` (Workbench), `.cdb` (mesh), `.rst`/`.rth` (results), `.inp` (APDL), `.out` (log) |
| CAE-6, 7 | COMSOL | `.mph` (model), `.mphbin` (binary), `.m`/`.java` (LiveLink scripts) |
| CAE-9 | fe-safe | `.fes` (model), `.csv` (life/damage export) |

**Cross-cutting (every eval):**
- Neutral exchange: `.step`/`.stp`, `.iges`/`.igs`, `.stl`
- 2D drawings: `.pdf`, `.dxf`, `.dwg`
- Evidence: `.png` (screenshots in `screenshots/`), `.mp4` (screen recording)
- Automation artifacts if the agent recorded one: `.swp`/`.sldmacro` (SolidWorks VBA), `.py` (Abaqus/ANSYS scripting), `.rpy`/`.jnl` (Abaqus replay/journal), `.mac` (APDL), `.jou` (NX journal)

**Practical rule:** the native project file in `output_files/` is authoritative (the "end file" the agent produced). STEP/PDF/PNG are derived artifacts for comparison against `expected/` and for evaluator review. `api_call_log.json` captures the sequence that produced them.

---

## Interpreting Expert Scores

After the expert evaluates a run, you receive a `score.yaml` (see EXPERT_GUIDE.md for format). Here is how to read it:

### Overall Verdict

| Verdict | What it means for you |
|---------|----------------------|
| **PASS** | Agent handled this task. No action needed unless latency is unusually high. |
| **PARTIAL** | Agent got most of it right but missed some criteria. Read the per-criterion notes to understand what went wrong. |
| **FAIL** | Agent could not do this task. Root-cause the failure mode before moving on. |

### Failure Modes — What to Do

| Failure Mode | What it tells you | Action |
|-------------|-------------------|--------|
| `wrong_approach` | Agent chose an incorrect method entirely | Check if the guide.md covers the correct approach. If not, add it. If it does, investigate why the agent didn't follow it. |
| `incomplete` | Agent stopped before finishing | Check if the agent hit a context window limit, tool error, or just stopped prematurely. May need better continuation logic. |
| `geometry_error` | Agent introduced invalid geometry | Check the specific tool calls that created the error. May need guardrails or validation steps in the tool. |
| `judgment_error` | Agent made a wrong engineering decision | This is the hardest to fix. Check if the guide.md has guidance that should have prevented this. If not, work with the expert to add it. |
| `tool_misuse` | Agent used the wrong tool or feature | May indicate a gap in the agent's tool knowledge. Check tool documentation exposure. |
| `hallucination` | Agent reported success on something it didn't do | Investigate the verification step. Agent may need explicit self-checking prompts. |

### Distance to Pass

The expert estimates "how many minutes would a human need to fix this output to reach PASS." Use this as a severity indicator:

- **< 15 minutes**: Agent was close. Likely a minor gap in the guide or a missed edge case.
- **15-60 minutes**: Agent got the general approach right but missed significant details. Guide probably needs expansion.
- **> 60 minutes**: Agent's output is not useful as a starting point. Fundamental capability gap.

---

## Incorporating Expert Feedback into the Agent

This is the most important part of your job. The eval → feedback → improvement loop is the whole point.

### Updating guide.md

When an expert identifies a failure caused by missing knowledge:

1. **Read the expert's evaluator_notes.md** — they will often describe exactly what the agent should have known.
2. **Draft new guide items** that encode this knowledge. Each item should be:
   - Specific (not "be careful with contacts" — instead "set contact stiffness to 0.1x default for rubber parts")
   - Actionable (the agent can directly apply it)
   - Justified (briefly explain why, so the agent can generalize)
3. **Get the expert to review** your draft guide items before merging.
4. **Re-run the eval** to verify the guide improvement actually helps.

### Updating spec.yaml

Specs may need updates when:

- An expert identifies that a success criterion is ambiguous (two evaluators disagree)
- A criterion is found to be unreasonable (no expert can consistently achieve it)
- The complexity ratings were wrong (recalibrate after expert validation)
- New success criteria are needed (expert found an important check that was missing)

**Always bump `version`** when changing success criteria or the prompt.

### Updating prompt.txt

Prompts should only change if:

- An expert determines the prompt is so vague that no reasonable engineer could identify the task
- The prompt accidentally reveals information that should require the agent to infer

**Do NOT make prompts more specific** because the agent failed. The agent failing on a vague prompt is a legitimate eval result — making the prompt clearer to help the agent pass defeats the purpose.

---

## Tracking Progress Over Time

### Run Comparison

For each eval, maintain a history of runs:

```
results/
  run-2026-04-15/    # First run
  run-2026-04-22/    # After guide improvement
  run-2026-05-01/    # After agent update
```

Compare across runs:

- Did the verdict improve (FAIL → PARTIAL → PASS)?
- Did latency decrease (faster)?
- Did the failure mode change (different problem, or same one)?
- Did the expert's distance-to-pass decrease?

### Dashboard Metrics

Track weekly:

| Metric | Scope | Target |
|--------|-------|--------|
| Pass@1 rate | Overall, per category | Increasing trend |
| Pass@1 by category | CAD-1 through CAD-10, CAE-1 through CAE-10 | Identify weakest categories |
| Latency (median minutes) | Per category | Decreasing trend |
| Infeasibility detection rate | NEG evals only | > 90% |
| Guide item count | Per eval | Increasing (captures more knowledge) |

---

## Adding New Evals

When adding a new eval (e.g., from a new expert questionnaire):

1. Use `EVAL_SPEC_TEMPLATE.yaml` as your starting point.
2. Work with the expert to fill in the spec — you handle the YAML structure, they handle the engineering content.
3. The expert writes `prompt.txt` (you may need to help them make it appropriately vague — they tend to over-specify).
4. The expert writes `guide.md` — this is their most valuable contribution. Push for specificity.
5. Procure input files (expert-provided > reconstructed > synthetic).
6. Expert validates by performing the task themselves.
7. Run the eval against the current agent build for a baseline.
8. Commit everything and update the eval count in README.md.

---

## Common Pitfalls

- **Don't optimize for the benchmark.** If you find yourself hardcoding eval-specific behavior into the agent, stop. The goal is general capability, not benchmark gaming.
- **Don't skip expert validation.** A spec you wrote without expert review will have blind spots. The expert validation step is not optional.
- **Don't ignore PARTIAL results.** A PARTIAL that stays PARTIAL across 5 runs is more concerning than a FAIL that becomes PARTIAL — the FAIL/PARTIAL transition shows progress, the stuck PARTIAL suggests a systematic gap.
- **Don't conflate speed with quality.** A PASS that takes 20 minutes is better than a FAIL that takes 5. Optimize for correctness first, latency second.
- **Don't change prompts to help the agent pass.** The temptation is real. Resist it.
