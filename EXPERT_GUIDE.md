# Expert Guide — Nexus Agent Benchmark

How to create evals, validate them, evaluate agent runs, and contribute engineering knowledge.

## Your Role

You are a CAD or CAE domain expert. Your engineering knowledge is the most valuable input to this benchmark. Your job is to:

1. **Create eval cases** from real-world engineering tasks
2. **Write prompts** that mirror how engineers actually ask for help
3. **Write guides** that encode your tribal knowledge
4. **Evaluate agent outputs** by inspecting the actual model/results
5. **Provide feedback** that developers can use to improve the agent

You are NOT expected to write code, YAML, or deal with infrastructure. A developer will pair with you to handle the technical packaging. You provide the engineering content.

---

## Creating an Eval

### Where Eval Cases Come From

Eval cases come from real engineering tasks — things you or your colleagues have actually done. We want tasks that:

| Criterion | Why |
|-----------|-----|
| Took 3+ hours when you did it | We care about long, judgment-heavy tasks — not button clicks |
| Required 10+ distinct operations in the software | Filters out trivial tasks |
| Involved real engineering decisions | Not just following a procedure, but making judgment calls |
| Can be reproduced with synthetic or anonymized input files | We need to be able to run the eval repeatedly |

We do NOT want:
- Simple 2-click operations ("apply a material", "run mesh")
- Tasks that are primarily data entry
- Tasks where the only hard part is knowing which button to click

### What the Expert Delivers

For each eval, you contribute three artifacts. A developer handles the YAML packaging and directory structure.

#### 1. Input File

The starting state: the CAD/CAE file that represents the task at t=0 — a part with DFM violations, an assembly that needs restructuring, a STEP import with defects, a model ready to be meshed, etc.

- Must match the `state_description` in the spec.
- Must be reproducible (synthetic or anonymized — no customer IP).
- Must open cleanly in the target tool version listed in the spec.

#### 2. Prompt (prompt.txt)

This is what the user types to the agent. It should sound like how you would actually ask a colleague for help — casual, brief, and incomplete.

**Good prompts** (vague, realistic):
- "This assembly is too detailed for thermal FEA — can you simplify it?"
- "I need a drawing package for this manifold, it's going to a CNC shop. Full GD&T."
- "The O-ring model keeps crashing around 35%. Figure out why — don't change the material."

**Bad prompts** (over-specified, unrealistic):
- "Remove all fillets under 2mm, suppress cosmetic features, retain thermal pads as separate bodies, merge connector plates if thermally connected, export as STEP AP214 with angular tolerance 0.01 rad..."
- "Set up a 3-step analysis: Step 1 bolt pretension 160kN using *PRE-TENSION SECTION, Step 2 pressure 15 MPa with end cap force P*A/8..."

Nobody talks like the bad examples. The agent needs to figure out the details from engineering knowledge — that is the capability we are measuring.

**Rules for prompts:**
- 1-4 sentences maximum
- Casual tone — you are talking to a skilled colleague, not writing a spec
- Give enough context to identify WHAT you need, not HOW to do it
- Include key constraints that would change the approach (e.g., "no side actions", "two-plate mold", "ASME VIII Div 2")
- Do NOT include specific procedures, parameter values, or step-by-step instructions
- Do NOT reference the guide — the agent receives the guide separately

#### 3. End-File Package

The reference answer. Two pieces, delivered together:

**(a) The end file itself.** Relative to the input file, both must be modified:
- **Model tree** — features added, removed, renamed, reordered, or suppressed as the task requires.
- **Geometry** — the physical changes the task required (draft applied, walls unified, undercuts eliminated, components regrouped into sub-assemblies, mesh applied with correct element type and size, BCs placed, etc.).

Both changes must be baked into the file. A written description or guide alone is not sufficient — the end file is the concrete reference the agent's output is compared against. If you cannot produce the end file in the target tool yourself, the eval is not well-posed.

**(b) Success criteria** — a list of 8-15 specific, verifiable checks. Each must be answerable as PASS / PARTIAL / FAIL by an evaluator looking at the output file.

**Good criteria** (specific, verifiable):
- "All fillets with radius < 2mm on non-thermal-path faces removed"
- "Cyclic symmetry BCs correctly applied on both cut faces (1/8 sector = 45 degree)"
- "Maximum von Mises stress in optimized result < 280 MPa in all 3 load cases"

**Bad criteria** (subjective, unverifiable):
- "Geometry looks reasonable"
- "Analysis is set up correctly"
- "Results are good"

**Mark criteria as `critical: true`** when failure on that criterion means the entire output is unusable. Examples:
- Introduced geometric errors (watertight body becomes non-watertight)
- Lost components during restructuring
- Agent reported safety when the analysis is invalid
- Wrong material properties in a structural analysis

---

### What the Developer Does Next

The expert's three artifacts define the test. The developer then runs it:

1. **Run the agent.** Provide `(input file + prompt)` to the agent. The agent produces an output file.
2. **Evaluate against success criteria.** Instruct an evaluator agent to open the output file and check it against each criterion from the end-file package, returning PASS / PARTIAL / FAIL per criterion. The reference end file is available to the evaluator for comparison.
3. **Screenshot generation (optional).** Render matched views of the end file and the output file so a human reviewer can visually confirm key criteria.
4. **Record.** Write results to `results/run-YYYY-MM-DD/score.yaml`.

---

### The Guide (shared eval-suite artifact)

In addition to the per-eval artifacts above, each eval category has a `guide.md` file encoding tribal knowledge — the things you would tell a new hire before letting them attempt tasks in this category. The guide is embedded in the agent's system prompt, not given to the user, and not part of any single test case's input.

**Good guide items** (specific, actionable, justified):
- "Thermal pads are sacred. Never merge or simplify them — their thickness drives thermal resistance (R = t/kA). Even 0.1mm change corrupts results."
- "Suppress features before deleting. SolidWorks feature trees have parent-child dependencies that aren't obvious — suppressing first lets you check for cascading failures."
- "Bolt pretension must use *PRE-TENSION SECTION with FIXED AT CURRENT LENGTH in subsequent steps — otherwise preload is lost when pressure is applied."

**Bad guide items** (generic, vague, obvious):
- "Be careful with the geometry"
- "Make sure the mesh is good"
- "Check the results"

**What makes a great guide item:**
1. **Specific** — names the exact feature, parameter, or operation
2. **Actionable** — the agent can directly apply it
3. **Justified** — briefly explains WHY (so the agent can generalize to similar situations)
4. **Non-obvious** — an engineer with 2 years of experience might not know this, but one with 10 years would

---

## Validating an Eval

Before any eval enters the active suite, you validate it. This is the quality gate.

### Validation Checklist

1. **Read the spec.** Does the task description match a real-world workflow you've encountered?
2. **Read the prompt.** Is it realistically vague? Would an engineer actually type this? If it reads like a specification, push back — make it vaguer.
3. **Read the guide.** Are the items specific enough to be useful? Would they actually help a junior engineer? Are any critical items missing?
4. **Check the input files.** Open them in the target tool. Does the starting state match the spec description? Are there unexpected issues?
5. **Perform the task yourself.** (Or confirm that the expected output state is correct.) This is the most important step. If you cannot complete the task to the success criteria, the eval is broken.
6. **Review the success criteria.** Are they complete? Is there a check that would let a bad result pass? Is there a check that would fail a good result?
7. **Sign off.** Confirm that you are satisfied the eval is accurate, the criteria are complete, and a PASS on all criteria genuinely means the task was done well.

### Common Issues to Catch

- **Over-specified prompt**: Push back. The prompt should not contain the solution.
- **Missing critical criterion**: If there is a way the agent could produce a dangerous or broken result that would still pass all criteria, add a criterion.
- **Unrealistic complexity rating**: Recalibrate based on your own experience.
- **Ambiguous criterion**: If two evaluators could reasonably disagree on PASS vs FAIL, the criterion needs to be more specific.

---

## Evaluating Agent Runs

When the developer runs an eval, you receive an evaluator packet and score the result.

### What You Receive

1. The eval spec (prompt, guide, success criteria)
2. The input files (starting state)
3. The agent's output files (what it produced)
4. Screenshots and model tree captures
5. Optionally: the API call log (for debugging)

### Step 1: Review the Success Criteria

Open `spec.yaml` and locate the `success_criteria` list. Each criterion has:

- `id`: Unique identifier (e.g., `SC-01`).
- `description`: What to check.
- `critical`: Whether failure on this criterion is an automatic FAIL for the entire eval.
- `type`: The kind of check (geometry, count, property, documentation, numerical).

### Step 2: Score Each Criterion

For each criterion, assign one of:

| Score | When to use |
|-------|------------|
| **PASS** | Criterion is fully met. You have no reservations. |
| **PARTIAL** | Criterion is partly met. Some elements correct, others missing or wrong. Counts as 0.5 for percentage calculation. |
| **FAIL** | Criterion is not met. |

### Step 3: Determine Overall Verdict

Calculate the pass percentage:

```
pass_pct = (num_pass + 0.5 * num_partial) / total_criteria
```

| Verdict | Condition |
|---------|-----------|
| **PASS** | All criteria met (pass_pct = 100%) |
| **PARTIAL** | pass_pct >= 70% AND no critical failures |
| **FAIL** | pass_pct < 70% OR any critical failure |

A **critical failure** is any criterion marked `critical: true` that scores FAIL. Critical failures override the percentage calculation — even if 90% of criteria pass, a single critical failure results in an overall FAIL.

### Step 4: Classify the Failure Mode

If the result is PARTIAL or FAIL, classify the primary failure:

- `wrong_approach` — agent chose an incorrect method entirely
- `incomplete` — agent stopped before finishing
- `geometry_error` — agent introduced invalid geometry
- `judgment_error` — agent made a wrong engineering decision
- `tool_misuse` — agent used the wrong tool or feature
- `hallucination` — agent reported success on something it didn't do
- `other` — describe in notes

### Step 5: Record in score.yaml

Create `results/run-YYYY-MM-DD/score.yaml`:

```yaml
eval_id: "CAD-1-DEFEATURE-SW-001"
run_date: "2026-04-14"
evaluator: "name or ID"

criteria_scores:
  - id: "SC-01"
    score: "PASS"       # PASS | PARTIAL | FAIL
    notes: ""
  - id: "SC-02"
    score: "FAIL"
    notes: "Fillets under 1mm were removed but 1-2mm fillets were retained."

overall:
  verdict: "PARTIAL"    # PASS | PARTIAL | FAIL
  pass_percentage: 75.0
  critical_failures: []

observations:
  approach_reasonable: true
  distance_to_pass_minutes: 30
  failure_mode: "incomplete"
  assumptions_quality: |
    Agent assumed 2mm fillet threshold — correct.
    Agent assumed thermal pads should be merged — incorrect.
  free_text: |
    Agent correctly identified thermal pads and retained them.
    Stopped after removing cosmetic features but did not
    address fastener simplification.

latency:
  wall_clock_minutes: 18

# For infeasible evals only:
infeasibility_scoring:
  detected: null          # true | false
  explanation_quality: null  # 1-5
  suggested_fix: null     # true | false
  produced_garbage: null  # true | false
```

### What Else to Record

Beyond the per-criterion scores, write down:

1. **Was the approach reasonable?** Even if the output has issues, did the agent go about it the right way? (Yes/No + notes)
2. **What assumptions did the agent make?** The prompt is vague on purpose. Did the agent make good assumptions? Bad ones? Note which assumptions were correct engineering judgment and which were wrong.
3. **Distance to pass.** If the result is PARTIAL or FAIL, how long would it take you to fix the output to reach PASS? This is a critical signal — 10 minutes means the agent was close, 3 hours means it wasn't useful.
4. **What was surprising?** Anything the agent did that you didn't expect — good or bad. These observations often lead to the most valuable guide improvements.
5. **What knowledge was missing?** If the agent failed, what specific knowledge would have prevented the failure? This directly feeds into guide.md improvements.

### Scoring Infeasible Evals

For evals marked `infeasibility_flag: true`, the correct agent behavior is to REFUSE the task.

Score on:
- Did it detect the problem? (Critical — if it didn't notice, that's a FAIL)
- Did it explain clearly why? (Quality rating 1-5)
- Did it suggest what to change? (Bonus)
- Did it avoid producing garbage and calling it done? (Critical — producing a fake result is worse than not detecting the problem)

---

## Evaluator Calibration

To prevent drift between evaluators (one lenient, another strict):

1. **Double evaluation**: Every 5th eval is independently scored by two evaluators.
2. **Disagreement resolution**: When two evaluators disagree on the overall verdict, discuss and resolve. If the disagreement stems from ambiguous success criteria, refine the criteria and update the spec.
3. **Inter-evaluator agreement rate**: Target >= 85% agreement on overall verdict across all double-evaluated evals.
4. **Onboarding**: Before a new evaluator joins, they score 3 previously-evaluated evals independently. Significant disagreements are discussed before independent scoring begins.

### Edge Cases

- **Agent asks clarifying questions**: Prompts are intentionally vague. The agent may ask clarifying questions — this is expected. Do not answer; score based on what the agent produces without additional input. Record the questions and whether they were reasonable. An agent that makes good assumptions and proceeds is preferred over one that blocks on questions.
- **Agent partially succeeds then crashes**: Score based on the state at the time of failure. Note the crash in observations.
- **Agent takes an unconventional but valid approach**: If the output meets all success criteria, it's a PASS regardless of method. Note the unexpected approach for future eval design.
- **Agent modifies input files in unexpected ways**: Check that no input geometry was corrupted. Flag it as a critical failure even if not explicitly listed in criteria.
- **You are unsure**: Mark the criterion as PARTIAL, explain the uncertainty in notes, and flag for discussion in the next calibration session.

---

## Providing Feedback That Improves the Agent

Your evaluator notes are the raw material for agent improvement. Here is how to make them maximally useful:

### Be Specific About What Went Wrong

Not useful: "The mesh was bad."

Useful: "The agent used global refinement instead of local refinement at the volute tongue. The tongue region had 2mm elements (same as everywhere else) but needs 0.3mm elements to resolve the stress gradient. This is a knowledge gap — any turbocharger meshing engineer would refine the tongue first."

### Name the Missing Knowledge

Not useful: "The agent didn't know how to do it."

Useful: "The agent didn't know that for Abaqus gasket elements, the *PRE-TENSION SECTION must be set to FIXED AT CURRENT LENGTH in subsequent steps. This is a common Abaqus-specific gotcha that should be in the guide."

### Distinguish Capability Gaps from Knowledge Gaps

- **Capability gap**: The agent cannot perform the required tool operations (e.g., it cannot drive SolidWorks' Insert Mold toolset). This requires tool development, not guide updates.
- **Knowledge gap**: The agent can perform the operations but makes wrong engineering choices (e.g., it removes thermal pads during defeaturing). This is fixable with guide improvements.

Label your observations accordingly — it helps developers prioritize.

### Suggest Guide Additions

When you identify a knowledge gap, draft the guide item yourself. You are the domain expert — the developer can format it, but the engineering content must come from you.

Format: "[What to do] — [Why, in one sentence]"

Example: "Set contact stiffness to 0.1x default for initial substeps in rubber analysis — default stiffness causes chattering at large deformations because the contact algorithm over-corrects."

---

## Writing New Evals from Expert Questionnaires

If you are contributing eval cases from the Expert Questionnaire process:

### Step 1: Identify Candidate Tasks

From your questionnaire responses, select tasks that meet the criteria (3+ hours, 10+ operations, real engineering judgment). For each candidate, write a brief (2-3 sentence) description of:
- What the task is
- Why it was hard
- What the correct outcome looks like

### Step 2: Pair with a Developer

A developer will help you package the task into the eval format. You provide:
- The engineering content (task description, success criteria, guide items)
- Input files (or enough information to reconstruct them)
- Reference output (or clear description of what "done correctly" looks like)

The developer handles:
- YAML formatting
- Directory structure
- Integration with the eval infrastructure

### Step 3: Write the Prompt as a User Would Say It

Imagine you are asking a colleague to do this task. What would you actually say? Write that. It should be 1-4 sentences.

If you find yourself writing more than 4 sentences, you are over-specifying. Move the extra detail into the guide.

### Step 4: Write the Guide as Advice to a Junior Engineer

Imagine a junior engineer with 2 years of experience is about to attempt this task for the first time. What do you tell them? Write those items.

Each item should save the junior engineer from a mistake they would otherwise make, or teach them a heuristic they would otherwise learn the hard way.

### Step 5: Define Success Criteria

For each criterion, ask yourself: "If I were reviewing this junior engineer's work, what would I check?" Each check becomes a success criterion.

Mark a criterion as critical if failure on it would make you reject the work entirely — not just send it back for revision, but reject it.

---

## FAQ

**Q: How vague should the prompt really be?**
A: As vague as you would be when asking a competent colleague. You don't explain fundamentals to a peer — you say what you need and trust them to figure out the how. If the prompt reads like a work instruction or SOP, it's too detailed.

**Q: What if the agent can't possibly succeed without information I left out of the prompt?**
A: That information should be in the guide.md, not the prompt. The guide is the agent's engineering knowledge. The prompt is the user's request. These are different things.

**Q: Can I reference standards in the prompt?**
A: Yes — engineers casually reference standards all the time ("ASME Y14.5", "BS 7608", "ASME VIII Div 2"). Standards are context, not instructions.

**Q: What if I disagree with another evaluator's score?**
A: Discuss it. If the disagreement comes from ambiguous criteria, the criteria need to be refined. If it comes from different engineering judgment, document both perspectives and the resolution. This calibration is part of the process.

**Q: How do I handle tasks where there are multiple correct approaches?**
A: Write success criteria about the OUTPUT, not the METHOD. If the result is correct, the approach doesn't matter. Note the approach in your observations for future reference, but don't fail an eval because the agent took a different path than you would have.

**Q: Should I create evals that I think the agent will fail?**
A: Yes. An eval suite where everything passes is useless. The benchmark should include tasks at the edge of the agent's capability. Failures are the most valuable signal — they tell us where to improve.
