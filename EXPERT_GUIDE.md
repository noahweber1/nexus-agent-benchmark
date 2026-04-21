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

> **Folder roles at a glance.** `input/`, `expected/`, and `results/` all hold the same kinds of files (e.g., a `.sldprt` can appear in each), but play different roles:
>
> - `input/` — the starting state as a **native program file** (e.g., `.sldprt`, `.cae`, `.mph`), handed to the agent along with `prompt.txt`. Authored once, immutable.
> - `expected/` — your reference answer as a **runnable reproduction bundle** centered on the native program file: the native file, a solver deck or journal (when the tool has one), the reference results, `tool_version.txt`, a short `REPRODUCE.md`, and a few screenshots. **You produce the native file by performing the task by hand inside the target software** — every click, every feature, every BC, every mesh control — exactly as you would if no agent existed. Authored once, immutable. Another engineer with the tool installed must be able to open the folder and reproduce the end state in under 30 minutes of wall-clock work, using only what is in the folder. See §End-File Package below for the per-tool artifact list.
> - `results/run-YYYY-MM-DD/` — an agent's actual attempt (again, the **native program file**) plus the evaluator's score. Grows over time, one subfolder per run, authored by the developer and evaluator (not you).
>
> You (the expert) produce the first two. The third is populated later when the eval is executed.
>
> **Why native, not STEP/IGES/STL?** The end state we are aiming for is what the target CAD/CAE program outputs — the file that preserves the parametric feature tree (fillets, extrudes, patterns, mates, BCs, study definitions). Neutral exports are dead geometry: fine as supplementary evidence, but they cannot serve as the reference answer because the model tree is gone.
>
> **Why by hand, not scripted?** The reference answer must capture the *engineering judgment* an experienced practitioner applies in the moment: which feature to suppress first, which face to fillet before the next, what element size is appropriate here, which BC set makes physical sense. A script or macro bakes in decisions that were made once and generalizes them away from the model at hand. Hand-driving the target software is what produces a reference answer that reflects real practitioner judgment — that is the thing the agent is being measured against.

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
- "Got a spring STEP from a vendor, `spring1.stp`, structural steel. Need compression and torsion stiffness for the datasheet." (see §Worked Example: Spring Stiffness for the full eval)

**Bad prompts** (over-specified, unrealistic, or adversarial):
- "Remove all fillets under 2mm, suppress cosmetic features, retain thermal pads as separate bodies, merge connector plates if thermally connected, export as STEP AP214 with angular tolerance 0.01 rad..."
- "Set up a 3-step analysis: Step 1 bolt pretension 160kN using *PRE-TENSION SECTION, Step 2 pressure 15 MPa with end cap force P*A/8..."
- "Import `C:\Users\me\Downloads\spring1.stp`, mesh with 2mm tetrahedra, fix bottom face, apply 100N axial, extract displacement, compute k=F/δ. DO NOT ASK ANY QUESTIONS. Reply with only two numbers." (procedure + absolute path + adversarial framing — all three disqualify it)

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

**(a) The reproduction bundle.** Not a single file — a set of artifacts in `expected/` that together let another engineer reproduce the end state by opening or executing them.

Relative to the input file, both must be modified in the native file:
- **Model tree** — features added, removed, renamed, reordered, or suppressed as the task requires.
- **Geometry** — the physical changes the task required (draft applied, walls unified, undercuts eliminated, components regrouped into sub-assemblies, mesh applied with correct element type and size, BCs placed, etc.).

Per-tool required artifacts:

| Tool | Runnable native | Recipe / deck (text) | Reference results | How you run it |
|---|---|---|---|---|
| SolidWorks | `.sldprt` / `.sldasm` / `.slddrw` | — | — | Open in SolidWorks; the feature tree rebuilds |
| ANSYS Workbench | **`.wbpz` archive** (not a bare `.wbpj` — a `.wbpj` is a pointer to an external directory tree and is useless on its own) | — | Embedded in the `.wbpz` if a Solution was computed | Open Workbench → File → Restore Archive → Update |
| ANSYS Mechanical (standalone) | `.mechdat` archive | `.dat` or `.inp` solver deck | `.rst` result file | Open the `.mechdat`; or run the deck with `mapdl -b -i input.dat` |
| ANSYS Fluent | `.cas.h5` + `.dat.h5` | `.jou` journal that rebuilds the case from the mesh | `.dat.h5` | `fluent 3ddp -g -t4 -i solve.jou` |
| LS-DYNA (via Workbench) | `.wbpz` + `.k` keyword deck | The `.k` deck — text and diffable | `d3plot` binaries | Open the `.wbpz` → Solution → Update; or standalone `lsdyna i=input.k` |
| Abaqus | `.cae` model database | **`.inp` text deck** — Abaqus's text input file is fully reproducible | `.odb` result file | `abaqus job=input cpus=N` |
| COMSOL | `.mph` with cleared solution | Optional `.class.java` Model Builder export | `.mph` with saved solution | Open in COMSOL → Compute |

Universal tail — every eval, every tool:

- **`tool_version.txt`** — exact version string (`SolidWorks 2024 SP3.0`, `ANSYS 2024 R2`, `Abaqus 2024 HF3`, `COMSOL 6.2`). Without this, a future reviewer cannot know which install to use.
- **`REPRODUCE.md`** — a short (5–10 line) plain-English recipe: open X → do Y → expect Z. Names every artifact the reviewer has to touch.
- **`geometry.step`** (CAD evals only) — supplementary neutral export. Forensic artifact only; never the reference.
- **3–5 reference screenshots** of key views — PNG files that let a reviewer without a license sanity-check the end state.

> **Reproduction bar.** Another engineer who has the tool installed must be able to open the `expected/` folder and reproduce the end state in under 30 minutes of wall-clock work, using only what is in the folder. If they have to email you to ask how it was built, the bundle is incomplete.

**How you produce the native file.** Open the `input/` file in the target software and **do the task by hand** — the same way you would complete it on a normal working day with no agent involved. Click through the feature tree, create the new features, suppress or delete what the task requires, apply the material or mesh controls, place the boundary conditions, run the solve, save. If the task takes four hours of hand work to complete correctly, then producing the reference file takes four hours. The native file must be the result of real practitioner work inside the application — not a scripted batch operation, not a macro replay, not a derivation from a STEP re-import. The product of that manual session is what the agent's output is compared against. The solver deck, results, and screenshots are then exported from that same session to complete the bundle.

A useful test: if you can describe every meaningful decision that went into the native file — *"I suppressed the fastener bores before applying fillets because otherwise the fillet tangent spilled into the bore edge"*, *"I picked a 2mm element size at the tongue because a 4mm size skipped the stress concentration"* — you probably built it the right way. If you can't recall making those decisions, the file may have come from an automated path that short-circuited the judgment the eval is meant to measure.

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

> **Reference.** See §Worked Example: Spring Stiffness at the end of this guide for a full eval — prompt, per-tool bundle, `fingerprint.yaml`, 8 success criteria with critical/non-critical flags, and a short decision journal. Use it as a calibration reference when deciding what your own eval's deliverables should look like.

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
5. **Perform the task yourself by hand inside the target software.** Open the input file in the tool, complete the task click-by-click as you would on a normal working day without an agent, and save the result as the `expected/` reference. This is the most important step. A written description of the end state, or a partially scripted equivalent, is not a substitute. If you cannot complete the task manually to the success criteria, the eval is broken.
6. **Review the success criteria.** Are they complete? Is there a check that would let a bad result pass? Is there a check that would fail a good result?
7. **Sign off.** Confirm that you are satisfied the eval is accurate, the criteria are complete, and a PASS on all criteria genuinely means the task was done well.

### Common Issues to Catch

- **Over-specified prompt**: Push back. The prompt should not contain the solution.
- **Missing critical criterion**: If there is a way the agent could produce a dangerous or broken result that would still pass all criteria, add a criterion.
- **Unrealistic complexity rating**: Recalibrate based on your own experience.
- **Ambiguous criterion**: If two evaluators could reasonably disagree on PASS vs FAIL, the criterion needs to be more specific.
- **Subjective numerical target**: If a criterion says "stress looks reasonable" instead of "peak von Mises within 5% of 278 MPa", tighten it. See §Worked Example for the shape of a fingerprint-anchored criterion set.

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

The Expert Questionnaire is the primary intake for new evals. This section explains exactly how each questionnaire field maps to an eval artifact, so you can work through a filled questionnaire and produce everything a developer needs.

### Step 1: Select Candidate Tasks

Not every task in a questionnaire becomes an eval. Read the full questionnaire and pick tasks that meet the bar:

- Section 1 Task reports 3+ hours of focused work
- Complexity ratings include at least one 4 or 5 in either *Geometric complexity* or *Engineering judgment required*
- Section 1 Q6 ("Where did you get stuck") describes tribal knowledge, not just button-clicking or UX complaints
- Section 1 Q7 artifacts can be anonymized and shared

Reject tasks that are primarily data entry, format conversion, or "find the right button in the UI." Reject tasks whose artifacts cannot be shared without violating confidentiality — recreating a synthetic equivalent is acceptable, but describing the task without files is not.

### Step 2: Map Questionnaire Fields to Eval Artifacts

This is the authoritative mapping. Work through every row for each selected task.

| Questionnaire Field | Eval Artifact | Notes |
|---|---|---|
| §0 Primary CAD/CAE tool | `spec.yaml.target_tool` | Exact tool name + version (e.g., "SolidWorks 2024"). |
| §0 Industry / application domain | `spec.yaml.application_domain` | Copy the checked items from the controlled list. |
| §0 Name | `spec.yaml.metadata.source_expert` | Initials or a hash if the expert prefers anonymity. |
| Task Template header (task name) | `spec.yaml.title` and the eval ID suffix | Short descriptive name, e.g. `CAD-3-DFM-SW-001`. |
| Task → Functional class | `spec.yaml.functional_class` | Copy the checked items verbatim. |
| Task Q1 (Starting State) | `spec.yaml.input_state.state_description` + file(s) under `input/` | Describe the state in the spec; place the anonymized file in `input/`. |
| Task Q2 (Intent) | `prompt.txt` | Do **not** copy Q2 verbatim — it will be too formal. Re-voice as 1-4 casual sentences (see Step 3). |
| Task Q3 (End State) | `spec.yaml.expected_output.state_description` + file(s) under `expected/` | The end file must have both model tree changes AND geometry changes baked in. |
| Task Q4 (Verification) | `spec.yaml.success_criteria[]` | Each distinct check becomes one `SC-N` entry (see Step 5). |
| Task Q5 (Complexity Rating) | `spec.yaml.complexity.{geometric, engineering_judgment, tool_proficiency}` | Copy the 1-5 ratings directly. `estimated_tool_calls` is derived separately. |
| Task Q6 (Where You Got Stuck) | Items in `guide.md` | The richest source of tribal knowledge. Each stuck point typically becomes one or two guide items (see Step 4). |
| Task Q7 (Artifacts) | Files under `input/`, `expected/`, and any shared macros/journals | Anonymize before committing. |
| §2 Workflow Patterns (A-E) | Candidates for future eval categories and shared category guides | Not a single eval — these responses inform the suite's coverage, not individual test cases. |
| §3 Task Mapping table row | Cross-check for the spec | Use it to confirm the per-task sections are consistent. |

### Step 3: Re-voice Q2 as the Prompt

Q2 ("Intent") is typically written in formal, complete sentences — that is how engineers describe tasks in a form, not how they ask for help. Re-voice it before committing to `prompt.txt`:

- Formal Q2: "Simplify the bracket assembly model by removing non-thermal features to enable tractable thermal finite element analysis."
- Casual prompt: "This assembly is too detailed for thermal FEA — can you simplify it?"

Rules (same as the Prompt section above): 1-4 sentences, no specific parameter values, no step-by-step instructions.

### Step 4: Extract Guide Items from Q6

Q6 ("Where did you get stuck or spend disproportionate time") is the richest source of tribal knowledge. For each bullet in Q6, ask:

- What did I learn that a junior engineer wouldn't know?
- What was the heuristic or rule I ended up applying?
- What would I have done differently from the start if I had known?

Each answer typically becomes one guide item. Use the format `[What to do] — [Why]`. Discard "the tool was slow" or "I had to click a lot" — those are UX complaints, not transferable knowledge.

### Step 5: Convert Q4 into Success Criteria

Q4 ("How did you know it was correct") often already reads like a checklist. Convert each distinct verification step into one criterion:

- Q4: "I visually inspected the geometry, checked that mass was within 30% of the original, and ran a draft analysis to confirm no faces were missing draft."
- Success criteria:
  - `SC-01`: Visual inspection shows no unintended geometry changes
  - `SC-02`: Defeatured mass is within 30% of original mass — `critical: true` (outside this window indicates structural geometry was lost)
  - `SC-03`: Draft analysis reports all faces with draft ≥ 1 degree

Mark a criterion as `critical: true` when a failure on it would cause the expert to reject the work entirely during review — not merely request revisions.

### Step 6: Produce the Input and End Files

The expert produces both files. A written description is not a substitute.

- **Input file** — the starting state, anonymized. If the original was proprietary, recreate a synthetic equivalent with the same relevant features and problems.
- **End file** — the correct answer, with both the model tree and the geometry modified as required by the task. **Produce it by completing the task manually inside the target software**, exactly as you would without an agent — open the input file, do the work click-by-click, save the native file. That hand-produced artifact is the reference; a scripted or derived equivalent is not.

If either file cannot be produced without violating confidentiality, the task is not a candidate.

### Step 7: Hand Off to a Developer

Provide the developer with:
- The filled questionnaire (or a draft `spec.yaml`)
- `prompt.txt`
- `guide.md`
- Input and end files under `input/` and `expected/`

The developer handles YAML formatting, directory structure, and integration with the eval infrastructure.

---

## Worked Example: Spring Stiffness (CAE)

This is a fully-formed eval that illustrates every deliverable the expert produces. Use it as a reference when you author your own evals — it shows the shape of a good prompt, the scope of a reasonable task, the engineering decisions the agent must make, and the numerical fingerprints that make the result objectively checkable.

### The task (one sentence)

A vendor ships a helical compression spring as STEP geometry. The engineer needs compression and torsion stiffness values for a datasheet.

### The prompt (what the user types)

```text
Got a spring STEP from a vendor, spring1.stp, structural steel.
Need compression and torsion stiffness for the datasheet.
```

Two sentences. Names the file, names the material, names the deliverable, names the business context (datasheet). Does not dictate load magnitude, element size, solver choice, BC scheme, or output format — all of that is engineering judgment the agent must exercise. This is what "casual, brief, and incomplete" looks like in practice.

### What the agent must do

1. Open `spring1.stp` in a CAE tool (ANSYS Mechanical, Abaqus, or equivalent)
2. Create two independent linear static studies: **compression** and **torsion**
3. Assign structural steel (E = 200 GPa, ν = 0.3 — the canonical values)
4. Identify the two end faces of the helical spring from the geometry
5. Mesh the volume with elements suitable for a slender helical body (3D solid, not shell or beam)
6. **Compression case:** fix one end face (all DOF), apply axial force at the other end face
7. **Torsion case:** fix one end face, apply axial torque at the other end face — distributed across the face via MPC/rigid coupling, not concentrated at a single point
8. Solve both studies
9. Extract total axial displacement at the loaded end (compression) and total rotation about the spring axis at the loaded end (torsion)
10. Compute `k_axial = F / δ_z` (N/mm) and `k_torsion = T / θ_rad` (N·m/rad)
11. Report both numbers with units

### Reference numbers (`fingerprint.yaml`)

```yaml
# Compression case
load_compression_N: 100
displacement_axial_mm: 0.006609
stiffness_axial_N_per_mm: 15131

# Torsion case
load_torsion_Nm: 1.0
rotation_axial_deg: 213
rotation_axial_rad: 3.7175
stiffness_torsion_Nm_per_rad: 0.269

# Mesh + material invariants
mesh_type: "C3D10 (quadratic tet)"
mesh_element_count_min: 10_000
material: "Structural steel (E=200 GPa, nu=0.3)"

# Pass tolerance
stiffness_tolerance_pct: 5.0
```

Key property: **stiffness is linear in applied load for a linear elastic analysis**. An agent that picks 10 N and 0.1 N·m should land on the same stiffness values within numerical tolerance. That invariance is why the fingerprint checks stiffness, not displacement — it lets the agent exercise judgment on the load magnitude without breaking the eval.

### Success criteria (`spec.yaml.success_criteria`)

```yaml
success_criteria:
  - id: SC-01
    description: "Axial stiffness within 5% of 15,131 N/mm"
    critical: true
  - id: SC-02
    description: "Torsional stiffness within 5% of 0.269 N·m/rad"
    critical: true
  - id: SC-03
    description: "Both spring ends identified as the two helical end-faces (not a mid-coil face, not a cut section)"
    critical: true
  - id: SC-04
    description: "Structural steel material assigned (E ∈ [195, 210] GPa, ν ∈ [0.28, 0.33])"
  - id: SC-05
    description: "Two independent static studies created — not a single combined load case"
  - id: SC-06
    description: "Torsional load applied as torque about the spring axis via distributed coupling, not a point tangential force"
  - id: SC-07
    description: "Mesh is 3D solid with at least 10,000 elements — not shell, not beam"
  - id: SC-08
    description: "Reported units are correct — N/mm for axial, N·m/rad for torsional"
```

Three `critical: true` criteria: the two stiffness numbers and the end-face identification. Wrong on any of those and the output is engineering-unusable. The other five are `critical: false` — E = 205 GPa instead of 200 GPa is a deviation but not a disaster.

### The `expected/` reproduction bundle

Per the per-tool matrix (§End-File Package):

```
expected/
├── spring_stiffness.wbpz          # Workbench archive — both studies solved
├── spring_stiffness.inp           # Abaqus text deck — text, diffable, CI-runnable
├── fingerprint.yaml               # the reference numbers above
├── environment.yaml               # tool version, cores used, units system
├── DECISIONS.md                   # why 100 N, why C3D10 at ~2 mm, why rigid MPC on the torsion end
├── REPRODUCE.md                   # 5-line recipe to reproduce
└── screenshots/
    ├── geometry_end_faces_labeled.png
    ├── mesh_overview.png
    ├── compression_deformed_shape.png
    └── torsion_deformed_shape.png
```

### `DECISIONS.md` excerpt (what it looks like)

```markdown
- **Picked 100 N compression and 1 N·m torsion.** Linearity makes the stiffness
  magnitude-independent, but a small load keeps rotations in the linear
  regime (the 213° I recorded is linear because there is no geometric
  stiffening in the model — keep it that way). Any load 10–500 N or
  0.1–5 N·m is fine.
- **C3D10 at ~2 mm on the coil.** Linear tets under-predict helical bending
  stiffness by 10-15%. Quadratic tets are cheap enough on this size part
  and capture the coil curvature accurately.
- **Rigid MPC on the torsion end face.** A point tangential force would
  produce a large local distortion at the load node and a wrong rotation
  reading. Coupling the face to a reference point and applying the torque
  at the RP gives a clean, face-averaged rotation.
- **Did NOT model contact between coils.** Compression load is small enough
  that coils do not touch — confirmed by checking the deformed shape. If
  the datasheet ever lists loads approaching coil-contact, this analysis
  needs a nonlinear contact pass.
```

### Why this example is a good shape

- **Short, casual prompt** — two sentences, no procedure baked in.
- **Real engineering judgment required** — which ends, how fine a mesh, how to distribute the torque load, which post-processing to use.
- **Objective outputs** — two scalars with units and a tolerance.
- **Fingerprint is magnitude-invariant** — linearity makes stiffness the robust invariant.
- **Mix of critical and non-critical criteria** — wrong stiffness fails; 5 GPa off in E does not.
- **Reproducible end-to-end** — `.inp` + `fingerprint.yaml` give a CI-verifiable bundle; `.wbpz` + screenshots give an evaluator a hand-verifiable bundle; `DECISIONS.md` gives a reviewer the "why."

When your own eval has this shape — vague prompt, judgment-heavy task, objective numerical outputs, a fingerprint the CI can check without a license — it is a well-posed eval. If it is missing any of those, it probably needs another pass.

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
