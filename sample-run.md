# Example: Optimizing a diagram-generator skill

This walkthrough shows a complete autoresearch run from start to finish.

## Setup

**Target skill:** `diagram-generator/SKILL.md` (~800 tokens)

**Test inputs:**
1. "OAuth flow diagram" — straightforward, happy path
2. "Microservices architecture with 8 services" — complex, many components
3. "Single-box diagram with a very long label" — edge case, tests overflow handling
4. "CI/CD pipeline" — known failure, previously produced numbered steps

**Eval criteria:**

```
EVAL 1: Text legibility
Question: Is all text fully legible with no truncated, overlapping, or cut-off words?
Pass: Every word is complete and readable
Fail: Any word is partially hidden, overlapping, or cut off

EVAL 2: Color palette
Question: Does the diagram use only soft/pastel colors — no neon, bright red, or high-saturation?
Pass: All fills and strokes are soft-toned
Fail: Any element uses a harsh or high-saturation color

EVAL 3: Linear layout
Question: Does the diagram flow in one direction (left-to-right or top-to-bottom)?
Pass: Clear directional flow with no scattered elements
Fail: Elements are arranged without a clear flow direction

EVAL 4: No numbering
Question: Is the diagram free of step numbers, ordinals, or sequential numbering?
Pass: No numbers used as labels or ordering
Fail: Any numbered step, ordinal, or sequence marker present
```

**Configuration:** 3 runs per experiment, budget cap of 8 experiments.

**Context budget:**
```
~800 (skill) + ~1,500 (avg output) + ~500 (scoring) = ~2,800 tokens/run
2,800 × 3 runs × 4 inputs ≈ 33,600 tokens/experiment
8 experiments × 33,600 ≈ 268,800 tokens total — feasible
```

## Baseline (experiment 0)

Ran the skill 3 times across all 4 inputs. 12 total outputs.

**Results:** 8/12 (66.7%)

| Eval | Pass rate | Common failure |
|---|---|---|
| Text legibility | 2/3 | Small text in complex diagrams, labels clipped at edges |
| Color palette | 2/3 | One diagram used bright red (#FF0000) for error states |
| Linear layout | 3/3 | All diagrams had clear directional flow |
| No numbering | 1/3 | Two diagrams had numbered steps ("Step 1", "Step 2") |

**Checkpoint:** "Baseline is 66.7%. The biggest failures are numbering (EVAL 4 at 1/3) and colors (EVAL 2 at 2/3). Planning to start with specific hex codes for the color palette — replacing the vague 'use pastel colors' instruction with actual values. Sound right?"

User confirmed.

## Experiment 1 — KEEP

**Score:** 10/12 (83.3%) | +16.6% from baseline

**Change:** Replaced the line "use soft pastel colors for all fills" with a specific palette:
```
Use ONLY these fill colors: #A8D8EA (light blue), #AA96DA (light purple), 
#FCBAD3 (light pink), #FFFFD2 (light yellow), #B5EAD7 (light green). 
For strokes, use the same colors darkened by 30%.
```

**Result:** Color eval went from 2/3 to 3/3. The bright red error state was replaced with light pink. Other evals unchanged — numbering still failing at 1/3.

## Experiment 2 — DISCARD

**Score:** 10/12 (83.3%) | No change

**Change:** Added anti-pattern: "Do NOT use red (#FF0000), orange (#FF8C00), or neon green (#39FF14) in any element."

**Result:** Zero improvement. The hex code palette from experiment 1 already eliminated harsh colors. This rule was redundant. Reverted to keep the skill simpler — unnecessary instructions add noise and consume tokens.

## Experiment 3 — KEEP

**Score:** 11/12 (91.7%) | +8.4% from previous best

**Checkpoint:** "Baseline 66.7% → now 83.3%. Kept 1, discarded 1. Still failing: numbering (1/3 runs on complex diagrams). Planning to add explicit anti-numbering instruction. Continue?"

User confirmed.

**Change:** Added to the anti-patterns section:
```
Never include step numbers, ordinal numbers (1st, 2nd, 3rd), or any 
sequential numbering in diagram labels. If the flow direction isn't 
obvious from the layout, use arrows — not numbers.
```

**Result:** Numbering eval improved from 1/3 to 2/3. The OAuth and CI/CD diagrams stopped using numbers. One microservices diagram still had numbered services ("Service 1", "Service 2") — likely because the input didn't name them, so the skill defaulted to numbering as a label strategy.

## Experiment 4 — KEEP

**Score:** 12/12 (100%) | +8.3% from previous best

**Change:** Added a worked example showing a correctly formatted diagram snippet:
```
Example — correct diagram labels:
  "Auth service" ✓  (descriptive name)
  "Step 1: Auth" ✗  (numbered)
  "Service 1"    ✗  (numbered placeholder)
  "Load balancer" ✓ (descriptive name)
```

**Result:** The worked example fixed the last failure. The microservices diagram now uses descriptive names ("API Gateway", "User Service", "Order Service") instead of numbered placeholders. All 12 outputs pass all 4 evals.

Hit 100% — stopping condition met (95%+ for 2 consecutive would have triggered, but we hit ceiling in one).

## Final delivery

```
Baseline:  8/12  (66.7%)
Final:    12/12  (100.0%)
Improvement: +33.3%
Experiments: 4 (3 kept, 1 discarded)
Keep rate: 75%
```

**Top changes (in order of impact):**
1. Specific hex code palette replaced vague "pastel" instruction — fixed all color failures
2. Worked example showing correct vs incorrect labels — fixed the last numbering failure  
3. Explicit anti-numbering instruction — fixed most numbering failures

**What didn't help:**
- Anti-pattern listing specific banned colors (redundant after hex palette was added)

**Remaining risks:**
- Test set is small (4 inputs). Could overfit. Recommend expanding to 6-8 inputs and re-running baseline to verify generalization before shipping.
- Very complex diagrams (12+ nodes) weren't tested. Text legibility may still fail at extreme density.
