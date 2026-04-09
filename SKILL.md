---
name: autoresearch
description: "Autonomously optimize any skill by running it repeatedly, scoring outputs against binary evals, mutating the prompt, and keeping improvements. Adapts Karpathy's autoresearch methodology to skill prompts. Use when: optimize this skill, improve this skill, run autoresearch on, make this skill better, self-improve skill, benchmark skill, eval my skill, run evals on, tune this prompt, tighten this skill. Also triggers for: why does my skill fail sometimes, my skill output is inconsistent, this skill works 70% of the time. Outputs: an improved SKILL.md, a results log, and a changelog of every mutation tried."
---

# Autoresearch for Skills

Most skills work about 70% of the time. The other 30% you get garbage. The fix isn't to rewrite the skill from scratch — it's to let an agent run it dozens of times, score every output, and tighten the prompt until that 30% disappears.

This skill adapts Andrej Karpathy's autoresearch methodology (autonomous experimentation loops) to skill prompts. Instead of optimizing ML training code, we optimize skill prompts.

---

## The core job

Take any existing skill, define what "good output" looks like as binary yes/no checks, then run an autonomous loop that:

1. Generates outputs from the skill using test inputs
2. Scores every output against the eval criteria
3. Mutates the skill prompt to fix failures
4. Keeps mutations that improve the score, discards the rest
5. Repeats until the score ceiling is hit or the user stops it

**Output:** An improved SKILL.md + `results.tsv` log + `changelog.md` of every mutation attempted.

---

## Before starting: gather context

**STOP. Do not run any experiments until all fields below are confirmed with the user. Ask for any missing fields before proceeding.**

1. **Target skill** — Which skill do you want to optimize? Need the exact path to SKILL.md.
2. **Test inputs** — What 3-5 different prompts/scenarios should we test the skill with? See the test input design section below for how to pick good ones.
3. **Eval criteria** — What 3-6 binary yes/no checks define a good output? See [references/eval-guide.md](references/eval-guide.md) for how to write good evals.
4. **Runs per experiment** — How many times should we run the skill per mutation? Default: 3. Higher = more reliable scores but burns more context. See the context budget section to calibrate this.
5. **Budget cap** — Max number of experiment cycles before stopping. Default: 10.

**After gathering eval criteria, audit them before proceeding.** Run each eval the user provides through the 3-question test from [references/eval-guide.md](references/eval-guide.md): (1) Would two different agents score the same output and agree? (2) Could a skill game this eval without actually improving? (3) Does this eval test something the user actually cares about? If any eval fails these checks — especially if it's vague ("looks nice"), subjective ("is engaging"), or uses a scale — push back. Explain why it's problematic and propose a specific binary alternative. Do not proceed with vague evals. Bad evals waste every experiment that follows.

### Designing good test inputs

Your 3-5 test inputs determine whether the skill genuinely improves or just games the specific cases you test. Bad test inputs produce a skill that aces the test and fails in production.

**Every test set should include:**

- **One straightforward case.** The "happy path" — the most common, typical use of the skill. If this doesn't pass, the skill is fundamentally broken.
- **One complex case.** A challenging but realistic scenario: longer input, more ambiguity, multiple constraints. This is where skills usually start failing.
- **One edge case.** Unusual format, minimal information, adversarial phrasing, or a scenario that sits at the boundary of what the skill should handle. This catches brittleness.
- **One case that's failed before** (if available). If you've seen the skill produce bad output for a specific input, include it. Known failure cases are the highest-signal test inputs you have.
- **One case from a different domain** (optional but recommended). If the skill is supposed to work across contexts (e.g., a writing skill used for both emails and blog posts), include a case from the less-tested domain to prevent overfitting.

**The diversity test:** Look at your test inputs together. If they all look similar — same length, same structure, same level of complexity — add variety. A test set of five easy inputs tells you nothing about hard cases.

---

## Context budget: the constraint most people miss

Every skill run consumes context. If your skill generates a 2,000-token output and you run it 3 times across 4 test inputs, that's ~24,000 tokens per experiment just for outputs — before counting the skill itself, eval reasoning, and mutation planning.

**Before starting, calculate your budget:**

```
tokens_per_run ≈ (skill length) + (avg output length) + (eval scoring overhead ~500 tokens)
tokens_per_experiment = tokens_per_run × runs_per_experiment × number_of_test_inputs
max_experiments ≈ (available context) / tokens_per_experiment
```

**Calibration guidelines:**

| Skill type | Typical output size | Recommended runs/experiment | Max test inputs |
|---|---|---|---|
| Light (tweets, configs, short code) | <500 tokens | 3 | 5 |
| Medium (emails, blog drafts, scripts) | 500-2000 tokens | 2-3 | 3-4 |
| Heavy (documents, presentations, long code) | 2000+ tokens | 2 | 3 |

**If context is tight, rotate test inputs.** Instead of running all 5 inputs every experiment, rotate: experiments 1-3 use inputs A, B, C; experiments 4-6 use inputs B, C, D; experiments 7-9 use inputs C, D, E. This gives you coverage across all inputs without running all of them every cycle. When rotating, only compare scores against experiments that used the same inputs.

**When running in Claude.ai (no subagents):** You're executing the skill yourself, which means you have full context from writing the skill. This makes self-scoring less reliable — you know what the "right" answer is. Compensate by being stricter with your evals and flagging any output where you're uncertain about the score. When context gets tight, summarize previous experiment results into a compact format (experiment number, score, one-line description of change) and drop the full outputs.

---

## Step 1: Read the skill

Before changing anything, read and understand the target skill completely.

1. Read the full SKILL.md file
2. Read any files in `references/` that the skill links to
3. Identify the skill's core job, process steps, and output format
4. Note any existing quality checks or anti-patterns already in the skill

Don't skip this. You need to understand what the skill does before you can improve it.

---

## Step 2: Build the eval suite

Convert the user's eval criteria into a structured test. Every check must be binary — pass or fail, no scales.

**Format each eval as:**

```
EVAL [number]: [Short name]
Question: [Yes/no question about the output]
Pass condition: [What "yes" looks like — be specific]
Fail condition: [What triggers a "no"]
```

**Rules for good evals:**
- Binary only. Yes or no. No "rate 1-7" scales. Scales compound variability and give unreliable results.
- Specific enough to be consistent. "Is the text readable?" is too vague. "Are all words spelled correctly with no truncated sentences?" is testable.
- Not so narrow that the skill games the eval. "Contains fewer than 200 words" will make the skill optimize for brevity at the expense of everything else.
- 3-6 evals is the sweet spot. More than that and the skill starts parroting eval criteria back instead of actually improving.

See [references/eval-guide.md](references/eval-guide.md) for detailed examples of good vs bad evals, including guidance on evaluating creative and subjective skills.

**Max score calculation:**
```
max_score = [number of evals] × [runs per experiment]
```

Example: 4 evals × 3 runs = max score of 12.

---

## Step 3: How scoring works

This is critical. Sloppy scoring produces false confidence — you think the skill improved when it didn't.

**The scoring protocol:**

For each output, score every eval independently. Before recording any score, write out your reasoning in a structured block:

```
OUTPUT [run]-[input]: Evaluating against EVAL [N]: [name]
Observation: [What you actually see in the output relevant to this eval]
Verdict: PASS / FAIL
Reason: [One sentence explaining the verdict]
```

**Rules for honest scoring:**

- **Score what you see, not what you intended.** If the output almost passes but has one flaw, it fails. "Close enough" is a fail.
- **Don't let the previous run's score anchor you.** Each output is scored independently. If run 1 passed and run 2 has the same issue but slightly less bad, it still fails.
- **When uncertain, fail it.** The purpose of the loop is to improve the skill. False passes hide problems; false fails just mean you try harder. Erring toward fail is safer.
- **Watch for the self-evaluation trap.** You wrote the mutation. You want it to work. This bias is real and unavoidable. Compensate by spending more time looking for reasons to fail than reasons to pass.

**For outputs you can't fully evaluate** (visual outputs, files that need rendering, outputs requiring domain expertise):
- Score the evals you *can* evaluate
- Mark unevaluable evals as SKIP with a reason
- Adjust the max score: `adjusted_max = (evaluable evals) × runs`
- Flag these to the user during the checkpoint (see step 5)

**Recording scores:**

After scoring all outputs in an experiment, produce the summary:

```
EXPERIMENT [N] SCORE SUMMARY
Eval 1 [name]: [X]/[runs] passed
Eval 2 [name]: [X]/[runs] passed
...
Total: [sum]/[max] ([percent]%)
```

---

## Step 4: Establish baseline

Run the skill as-is before changing anything. This is experiment 0.

1. Create a working directory: `autoresearch-[skill-name]/` (in the skill's folder if writable, otherwise in `/home/claude/`)
2. Copy the original SKILL.md as `SKILL.md.baseline`
3. Create `results.tsv` with the header row
4. Run the skill using the test inputs for the configured number of runs
5. Score every output against every eval using the scoring protocol from step 3
6. Record the baseline score

**results.tsv format (tab-separated):**

```
experiment	score	max_score	pass_rate	status	description	failing_evals
0	8	12	66.7%	baseline	original skill — no changes	text_legibility:2/3,color_palette:1/3
```

The `failing_evals` column tracks which evals fail most. This is the signal you use to decide what to mutate first.

---

## Step 5: Checkpoint — confirm before continuing

**After establishing baseline, STOP and confirm with the user before proceeding.**

Present:
1. The baseline score and what it means
2. Which evals are failing most and a brief description of the failure pattern
3. Your planned first mutation and why you think it'll help
4. Whether any evals were unevaluable (and why)

**Why this matters:** If the baseline evals are poorly written — testing the wrong things, too vague to score consistently, or misaligned with what the user actually cares about — you'll waste every subsequent experiment optimizing for the wrong target. This checkpoint catches that before it costs you.

Wait for the user to confirm one of:
- "Looks good, proceed" → start the experiment loop
- Adjustments to the evals → revise and re-run baseline
- Adjustments to the test inputs → revise and re-run baseline
- "Baseline is already good enough" → skip optimization, deliver as-is

**Second checkpoint at experiment 3.** After 3 experiments, present a brief progress update:

```
Progress after 3 experiments:
Baseline: [X]% → Current best: [Y]%
Kept: [N] mutations, Discarded: [N] mutations
Still failing: [list of persistently failing evals]
Plan for next 3: [brief description]
Continue? (or redirect me)
```

If the user doesn't respond within the same turn, continue — but if the score hasn't moved after 3 experiments, pause and reassess your approach before burning more cycles.

---

## Step 6: Run the experiment loop

This is the core autoresearch loop.

**Each cycle:**

1. **Analyze failures.** Look at which evals are failing most. Read the actual outputs that failed. Identify the pattern — is it a formatting issue? A missing instruction? An ambiguous directive?

2. **Form a hypothesis.** Pick ONE thing to change. Don't change 5 things at once — you won't know what helped.

   Good mutations:
   - Add a specific instruction that addresses the most common failure
   - Reword an ambiguous instruction to be more explicit
   - Add an anti-pattern ("Do NOT do X") for a recurring mistake
   - Move a buried instruction higher in the skill (priority = position)
   - Add or improve an example that shows the correct behavior
   - Remove an instruction that's causing over-optimization for one eval at the expense of others
   - Simplify — if a section isn't contributing to passing evals, cut it

   Bad mutations:
   - Rewriting the entire skill from scratch
   - Adding 10 new rules at once
   - Making the skill longer without a specific reason
   - Adding vague instructions like "make it better" or "be more creative"
   - Adding an instruction that just restates an eval criterion verbatim (the skill will learn to parrot it back without actually improving)

3. **Make the change.** Edit SKILL.md with ONE targeted mutation.

4. **Run the experiment.** Execute the skill with the test inputs for the configured number of runs.

5. **Score it.** Run every output through every eval using the scoring protocol. Calculate total score.

6. **Decide: keep or discard.**
   - Score improved → **KEEP.** Log it. This is the new baseline for comparison.
   - Score stayed the same → **DISCARD.** Revert SKILL.md to previous version. The change added complexity without benefit.
   - Score got worse → **DISCARD.** Revert SKILL.md to previous version.

7. **Log the result** in results.tsv and changelog.md.

8. **Check stopping conditions** (see below). If not met, go back to step 1.

**Stopping conditions — stop the loop when ANY of these is true:**
- The user manually stops you
- You hit the budget cap
- You hit 95%+ pass rate for 2 consecutive experiments
- Score hasn't improved for 3 consecutive experiments (you're stuck — see "when you're stuck" below)

**When you're stuck (3 experiments with no improvement):**

Before giving up:
- Re-read the failing outputs from scratch. Are you misdiagnosing the problem?
- Try the opposite approach — if you've been adding instructions, try removing one
- Try combining two near-miss mutations that were each discarded individually
- Try a structural change: reorder sections, change the example, revise the anti-patterns
- If nothing works after 2 more attempts, stop and report to the user. The remaining failures might need different evals, not a different prompt.

---

## Step 7: Write the changelog

After each experiment (whether kept or discarded), append to `changelog.md`:

```markdown
## Experiment [N] — [KEEP/DISCARD]

**Score:** [X]/[max] ([percent]%) | Delta from baseline: [+/-X]%
**Change:** [One sentence describing what was changed]
**Hypothesis:** [Why this change was expected to help]
**Result:** [What actually happened — which evals improved/declined]
**Failing outputs:** [Brief description of what still fails, if anything]
**Context used:** ~[X]k tokens this experiment
```

This changelog is the most valuable artifact. It's a research log that any future agent (or smarter future model) can pick up and continue from. The context usage line helps future runs calibrate their budget.

---

## Step 8: Deliver results

When the loop stops, present:

1. **Score summary:** Baseline → Final (percent improvement)
2. **Total experiments run:** How many mutations were tried
3. **Keep rate:** How many mutations were kept vs discarded
4. **Top changes that helped** (from the changelog)
5. **Remaining failure patterns** (what the skill still gets wrong and why)
6. **The improved SKILL.md** (already saved in place)
7. **Recommendation:** Whether further optimization would help, or whether the remaining failures need different evals or a fundamentally different approach

---

## Output format

The skill produces these files in `autoresearch-[skill-name]/`:

```
autoresearch-[skill-name]/
├── results.tsv          # score log for every experiment
├── changelog.md         # detailed mutation log with context usage
└── SKILL.md.baseline    # original skill before optimization
```

Plus the improved SKILL.md saved back to its original location.

**results.tsv example:**

```
experiment	score	max_score	pass_rate	status	description	failing_evals
0	8	12	66.7%	baseline	original skill — no changes	text_legibility:1/3,color_palette:1/3
1	10	12	83.3%	keep	added explicit hex codes replacing vague "pastel" instruction	color_palette:0/3
2	10	12	83.3%	discard	added anti-pattern for neon colors — no improvement over hex codes	(none new)
3	11	12	91.7%	keep	added worked example showing correct label formatting	text_legibility:0/3
4	12	12	100%	keep	moved "no numbering" instruction above the layout section	(none)
```

---

## Example: optimizing a diagram-generator skill

**Context gathered:**
- Target skill: `diagram-generator/SKILL.md` (~800 tokens)
- Test inputs: "OAuth flow diagram" (straightforward), "microservices architecture with 8 services" (complex), "single-box diagram with a very long label" (edge case), "CI/CD pipeline" (previously failed — produced numbered steps)
- Evals: (1) All text legible? (2) Only pastel/soft colors? (3) Linear layout? (4) No numbering?
- Runs per experiment: 3, Budget cap: 8 experiments
- Context budget: ~800 + ~1500 avg output + ~500 scoring = ~2800/run × 3 runs × 4 inputs ≈ 33,600 tokens/experiment. Budget of 8 experiments = ~270k tokens. Feasible.

**Baseline (experiment 0): 8/12 (66.7%)**
Common failures: 2 diagrams had numbered steps, 1 had bright red, 1 had illegible small text.

**Checkpoint with user:** "Your skill scores 67% out of the box. The biggest problems are numbering (EVAL 4 fails 2/3 runs) and colors (EVAL 2 fails 1/3). I want to start by adding specific hex codes for the color palette. Sound right?" → User confirms.

**Experiment 1 — KEEP (10/12, 83.3%):**
Change: Replaced vague "pastel colors" with hex codes: `#A8D8EA, #AA96DA, #FCBAD3, #FFFFD2, #B5EAD7`.
Result: Color eval went from 2/3 to 3/3. Numbering still failing.

**Experiment 2 — DISCARD (10/12, 83.3%):**
Change: Added anti-pattern "Do NOT use red, orange, or neon green."
Result: No improvement — the hex codes from experiment 1 already solved colors. Reverted to keep the skill simpler.

**Experiment 3 — KEEP (11/12, 91.7%):**
Checkpoint: "Baseline 67% → now 83%. Kept 1, discarded 1. Still failing: numbering (1/3 runs). Plan: add explicit anti-numbering instruction. Continue?" → User confirms.
Change: Added "Never include step numbers, ordinals, or sequential numbering in diagrams."
Result: Numbering eval improved from 1/3 to 2/3. One stubborn case remains.

**Experiment 4 — KEEP (12/12, 100%):**
Change: Added worked example showing a correct diagram with labels but no numbers.
Result: Hit 12/12. All evals passing.

**Delivery:**
- Baseline: 8/12 (66.7%) → Final: 12/12 (100%)
- 4 experiments, 3 kept, 1 discarded (75% keep rate)
- Top changes: specific hex codes, anti-numbering instruction, worked example
- No remaining failures
- Recommendation: Skill is performing well on test inputs. Consider expanding the test set to 6-8 inputs and re-running baseline to check for overfitting before shipping.

---

## How this connects to other skills

**What feeds into autoresearch:**
- Any existing skill that needs optimization
- User-defined eval criteria (or help them define evals using the eval guide)

**What autoresearch feeds into:**
- The improved skill replaces the original
- The changelog can be passed to future models for continued optimization
- The eval suite can be reused whenever the skill is updated
- The results.tsv becomes training data for understanding which mutation types work best for which skill types

---

## The test

A good autoresearch run:

1. **Started with a baseline** — never changed anything before measuring
2. **Used binary evals only** — no scales, no vibes, no "rate 1-10"
3. **Scored honestly** — used the structured scoring protocol, erred toward fail when uncertain
4. **Changed one thing at a time** — so you know exactly what helped
5. **Kept a complete log** — every experiment recorded with context usage
6. **Checked in with the user** — at baseline and after experiment 3
7. **Improved the score** — measurable improvement from baseline to final
8. **Stayed within context budget** — didn't blow up at experiment 3 and lose state
9. **Didn't overfit** — the skill got better at the actual job, not just at passing the specific test inputs

If the skill "passes" all evals but the actual output quality hasn't improved — the evals are bad, not the skill. Go back to step 2 and write better evals.
