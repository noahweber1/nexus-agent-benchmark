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
| `expected/` | Reference screenshots, expected model tree, outputs | Expert provides | Evaluator |
| `results/` | Agent outputs + evaluator scores (per run) | Developer records, Expert scores | Both |

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
