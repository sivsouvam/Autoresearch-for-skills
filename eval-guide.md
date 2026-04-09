# Eval Guide

How to write eval criteria that actually improve your skills instead of giving you false confidence.

---

## The golden rule

Every eval must be a yes/no question. Not a scale. Not a vibe check. Binary.

Why: Scales compound variability. If you have 4 evals scored 1-7, your total score has massive variance across runs. Binary evals give you a reliable signal — either the output has the quality or it doesn't.

---

## Good evals vs bad evals

### Text/copy skills (newsletters, tweets, emails, landing pages)

**Bad evals:**
- "Is the writing good?" → too vague, every scorer defines "good" differently
- "Rate the engagement potential 1-10" → scale, unreliable across runs
- "Does it sound like a human?" → subjective, will almost always score "yes"

**Good evals:**
- "Does the output contain zero phrases from this banned list: [game-changer, here's the kicker, the best part, level up, at the end of the day, it's worth noting]?" → binary, checkable against a specific list
- "Does the opening sentence reference a specific time, place, person, or sensory detail — not a generic statement?" → binary, structural
- "Is the output between 150-400 words?" → binary, measurable
- "Does it end with a specific CTA that tells the reader exactly what action to take?" → binary, structural

### Visual/design skills (diagrams, images, slides)

**Bad evals:**
- "Does it look professional?" → subjective
- "Rate the visual quality 1-5" → scale
- "Is the layout good?" → vague

**Good evals:**
- "Is all text in the image legible with no truncated, overlapping, or cut-off words?" → binary, specific
- "Does the color palette use only soft/pastel tones with no neon, bright red, or high-saturation colors?" → binary, checkable
- "Is the layout linear — flowing either left-to-right or top-to-bottom with no scattered elements?" → binary, structural
- "Is the image free of numbered steps, ordinals, or sequential numbering?" → binary, specific

### Code/technical skills (code generation, configs, scripts)

**Bad evals:**
- "Is the code clean?" → subjective, every engineer disagrees on what "clean" means
- "Does it follow best practices?" → vague — which best practices?

**Good evals:**
- "Does the code run without errors when executed?" → binary, testable (actually run it if you can)
- "Does the output contain zero TODO, FIXME, or placeholder comments?" → binary, greppable
- "Are all function and variable names descriptive (no single-letter names except i/j/k loop counters)?" → binary, checkable
- "Does the code include error handling for all external calls (API, file I/O, network)?" → binary, structural

### Document skills (proposals, reports, decks)

**Bad evals:**
- "Is it comprehensive?" → compared to what?
- "Does it address the client's needs?" → too open-ended to score consistently

**Good evals:**
- "Does the document contain all required sections: [list the specific sections]?" → binary, structural
- "Is every claim backed by a specific number, date, or source?" → binary, checkable
- "Is the document under [X] pages/words?" → binary, measurable
- "Does the executive summary fit in 3 sentences or fewer?" → binary, countable

### Creative/subjective skills (brand voice, storytelling, humor, tone)

This is the hardest category to eval. The instinct is to write evals like "does it match our brand voice?" — but that's unanswerable by an agent. The trick is to decompose subjective qualities into their observable components.

**Bad evals:**
- "Does it sound like our brand?" → subjective, inconsistent
- "Is the writing voice engaging?" → vague, will almost always pass
- "Would the target audience like this?" → unanswerable

**Good evals — decompose the subjective quality into signals:**

*For brand voice:*
- "Does the opening use a concrete noun or specific reference — not an abstract concept like 'innovation' or 'excellence'?" → tests for the specificity that strong brand voices share
- "Does the piece avoid every phrase on the brand's banned list: [list them]?" → binary, checkable against a real artifact
- "Does the tone match the reference sample? Specifically: is the sentence length distribution similar (mix of short and long, not all medium)?" → binary, structural

*For storytelling:*
- "Does the piece include at least one specific detail (a name, a date, a place, a sensory observation) in the first two sentences?" → tests for concreteness, which is what people actually mean by "engaging"
- "Does the narrative follow a single thread from start to finish — no topic jumps or tangents that don't resolve?" → binary, structural
- "Does the piece end differently from how it began — showing change, resolution, or a shift in perspective?" → tests for narrative arc

*For humor:*
- "Does the piece contain at least one instance of subverted expectation (setup then twist)?" → structural pattern that underlies most humor
- "Is every joke or comedic beat relevant to the subject — no random non-sequiturs?" → binary, checkable

*For writing tone (formal, casual, technical, etc.):*
- "Are contractions used consistently throughout (casual) / Are contractions absent throughout (formal)?" → binary, measurable proxy for tone
- "Does the piece address the reader directly with 'you' at least twice (conversational) / never (academic)?" → binary, countable

**The decomposition method:** When you have a subjective quality to eval, ask: "What would I point to in the output to prove it has this quality?" Those concrete things you'd point to become your evals. If you can't point to anything specific, the quality is too vague to optimize for — you'll need the user to define it more precisely.

---

## Common mistakes

### 1. Too many evals
More than 6 evals and the skill starts gaming them — it optimizes for passing the test instead of producing good output. Like a student who memorizes answers without understanding the material.

**Fix:** Pick the 3-6 checks that matter most. If everything passes those, the output is probably good.

### 2. Too narrow/rigid
"Must contain exactly 3 bullet points" or "Must use the word 'because' at least twice" — these create skills that technically pass but produce weird, stilted output.

**Fix:** Evals should check for qualities you care about, not arbitrary structural constraints. Ask: "If someone added this quality to an otherwise bad output, would the output become good?" If no, the eval is testing the wrong thing.

### 3. Overlapping evals
If eval 1 is "Is the text grammatically correct?" and eval 4 is "Are there any spelling errors?" — these overlap. A grammar fail often includes spelling. You're double-counting failures and making the score less informative.

**Fix:** Each eval should test something distinct. If two evals always pass or fail together across multiple runs, merge them or drop one.

### 4. Unmeasurable by an agent
"Would a human find this engaging?" — an agent can't reliably answer this. It'll say "yes" almost every time because it defaults to being agreeable about its own output.

**Fix:** Translate subjective qualities into observable signals. "Engaging" might mean: "Does the first sentence contain a specific claim, story, or question (not a generic statement)?" See the creative/subjective section above for the decomposition method.

### 5. Gaming-prone evals
"Contains fewer than 200 words" — this is technically measurable, but the skill will optimize for brevity at the expense of everything else. Any eval with a simple numeric threshold is prone to gaming.

**Fix:** Pair constraint evals with quality evals. If you have a word-count eval, also have an eval that checks for completeness or depth. The two together prevent gaming.

---

## Writing your evals: the 3-question test

Before finalizing an eval, ask:

1. **Could two different agents score the same output and agree?** If not, the eval is too subjective. Decompose it into observable signals.
2. **Could a skill game this eval without actually improving?** If yes, the eval is too narrow. Broaden it or pair it with a complementary eval.
3. **Does this eval test something the user actually cares about?** If not, drop it. Every eval that doesn't matter dilutes the signal from evals that do.

---

## Template

Copy this for each eval:

```
EVAL [N]: [Short name]
Question: [Yes/no question]
Pass: [What "yes" looks like — one sentence, specific]
Fail: [What triggers "no" — one sentence, specific]
```

Example:

```
EVAL 1: Text legibility
Question: Is all text in the output fully legible with no truncated, overlapping, or cut-off words?
Pass: Every word is complete and readable without guessing
Fail: Any word is partially hidden, overlapping another element, or cut off at the edge
```

Example (creative/subjective):

```
EVAL 3: Opening specificity
Question: Does the opening sentence reference a specific detail — a name, place, time, number, or sensory observation?
Pass: The first sentence contains at least one concrete, specific reference (not abstract concepts like "innovation" or "the future")
Fail: The first sentence is generic and could apply to any topic without modification
```
