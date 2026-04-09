# Autoresearch for Skills

Most skills work about 70% of the time. The other 30% you get garbage.

The fix isn't to rewrite the skill from scratch. It's to run it dozens of times, score every output against binary evals, mutate the prompt, and keep what works. Discard what doesn't.

This is [Andrej Karpathy's autoresearch methodology](https://x.com/karpathy/status/1886192184808149383) adapted for Claude Code skills. Instead of optimizing ML training code, we optimize skill prompts.

## What it does

You point it at any existing skill. Define what "good output" looks like as yes/no checks. Then it runs an autonomous loop:

1. Runs the skill against test inputs
2. Scores every output against your eval criteria
3. Makes one targeted change to the skill prompt
4. Keeps the change if the score improves, reverts if it doesn't
5. Repeats until the score ceiling is hit

You get back: an improved SKILL.md, a `results.tsv` log of every experiment, and a `changelog.md` documenting every mutation tried — kept or discarded.

## Install

Drop the skill into your Claude Code skills directory:

```bash
# Option 1: Clone and copy
git clone https://github.com/user/autoresearch.git
cp -r autoresearch ~/.claude/skills/

# Option 2: Download the .skill package
# Download autoresearch.skill from releases
# Then install via Claude Code
```

The skill auto-triggers when you say things like:
- "Optimize my diagram-generator skill"
- "Run autoresearch on my technical-writing skill"  
- "My resume skill works 70% of the time, tighten it up"
- "Run evals on my email-writer skill"

## How it works

### The loop

```
Read skill → Establish baseline → [Analyze failures → Hypothesize → 
Mutate ONE thing → Run → Score → Keep or discard → Log] → Deliver results
```

### What makes it different from just "iterate on your prompt"

**Binary evals only.** No "rate this 1-10." No vibes. Every check is yes/no. Scales compound variability. Binary gives you a reliable signal.

**One mutation at a time.** Change five things at once and you'll never know which one helped. Each experiment changes exactly one thing. If the score improves, you know why.

**Context budget awareness.** Every skill run eats context tokens. The skill calculates how many experiments you can afford before you start, and adjusts runs-per-experiment based on your skill's output size. No more blowing up at experiment 3 and losing all state.

**Eval-quality validation.** Users provide bad evals. "Looks nice" and "is clean" aren't testable. The skill audits every incoming eval against a 3-question test and pushes back with specific binary alternatives before wasting cycles on vague criteria.

**Checkpoints, not blind autonomy.** Stops after baseline to confirm evals are aligned. Checks in again after experiment 3. Catches misaligned optimization early instead of burning 10 cycles on the wrong target.

**Complete research log.** The changelog documents every mutation — what changed, why, what happened, what still fails. Any future agent (or smarter future model) can pick it up and continue.

### The eval guide

The `references/eval-guide.md` file covers how to write good binary evals across five skill categories:

- **Text/copy** (newsletters, emails, landing pages)
- **Visual/design** (diagrams, slides, images)
- **Code/technical** (scripts, configs, code generation)
- **Documents** (reports, proposals, decks)
- **Creative/subjective** (brand voice, storytelling, humor, tone)

The creative/subjective section is the one most eval guides skip. It includes a decomposition method: take a subjective quality like "matches our brand voice" and break it into observable, scorable signals.

## File structure

```
autoresearch/
├── SKILL.md              # The skill prompt (this is what Claude reads)
├── references/
│   └── eval-guide.md     # How to write good binary evals
├── examples/
│   └── sample-run.md     # Walkthrough of a real optimization run
├── LICENSE
└── README.md             # You're here
```

## Example run

Here's a condensed version of optimizing a diagram-generator skill:

```
Baseline:  8/12  (66.7%)  — numbered steps, bright colors, illegible text
Exp 1:    10/12  (83.3%)  KEEP — replaced "pastel colors" with specific hex codes
Exp 2:    10/12  (83.3%)  DISCARD — anti-neon rule redundant after hex codes
Exp 3:    11/12  (91.7%)  KEEP — added "never include step numbers" instruction
Exp 4:    12/12  (100%)   KEEP — added worked example of correct formatting
```

4 experiments. 3 kept, 1 discarded. Baseline 66.7% to final 100%. The full walkthrough is in `examples/sample-run.md`.

## Dogfooding

We ran autoresearch on the autoresearch skill itself. Results:

| Experiment | Score | Status | What changed |
|---|---|---|---|
| 0 (baseline) | 80% | — | Original skill, no changes |
| 1 | 100% | KEEP | Added eval-quality validation gate |
| 2 | 100% | KEEP | Expanded test coverage to all 5 evals |
| 3 | 100% | KEEP | Adversarial stress test — contradictory evals caught |

One actual skill mutation was needed (the eval validation gate). The rest confirmed the core loop instructions were already solid. Full results in the [autoresearch run log](examples/dogfood-changelog.md).

## Design decisions

**Why binary evals instead of scales?** A 4-eval rubric scored 1-7 has a theoretical range of 4-28. In practice, scores cluster around 18-22 with high variance. You can't tell if a 20→21 change was signal or noise. Binary (pass/fail) gives you a denominator you trust: 14/20 means 14 outputs passed, period.

**Why one mutation per experiment?** Same reason as A/B testing. If you change the headline AND the button color AND the price, you can't attribute the conversion lift. Skill optimization works the same way.

**Why checkpoints instead of full autonomy?** The original v1 skill said "NEVER STOP." That sounds productive but it means if your evals are misaligned, you waste every cycle. A 30-second checkpoint after baseline catches bad evals before they cost you. The skill still runs autonomously between checkpoints.

**Why no live dashboard?** Earlier versions generated a Chart.js dashboard. In practice, the dashboard consumed context tokens, required CDN access, and couldn't stay live in Claude.ai's execution model. A clean TSV log + post-run summary gives you the same information without the overhead.

## Known limitations

- **Self-evaluation bias.** The same model that generates the output also scores it. The skill warns about this and provides a structured scoring protocol to reduce bias, but doesn't mechanically prevent it. A separate eval agent would be better — but requires subagent support.
- **Context window ceiling.** Heavy skills (2000+ token outputs) limit you to 2 runs per experiment with 3 test inputs. The skill includes a budget calculator, but you'll hit diminishing returns around 6-8 experiments for heavy skills.
- **Can't test multi-turn behavior.** Skills that require back-and-forth with the user (interview-style, clarification loops) can't be fully exercised in the autoresearch loop. You can test the first turn, but not the full conversation.

## Contributing

PRs welcome for:
- New eval examples in the eval guide (especially for skill categories not covered)
- Context budget optimizations
- Subagent-based scoring for reducing self-eval bias
- Integration with Claude Code's native skill testing infrastructure

## Credits

Methodology adapted from [Andrej Karpathy's autoresearch concept](https://x.com/karpathy/status/1886192184808149383) — autonomous experimentation loops for ML research, applied here to prompt engineering.

## License

MIT
