# Claude Code Prompt: GBrain README Update for v0.4 Release

## Context

GBrain v0.4.0 just shipped. The big addition besides the technical features (doctor command, parallel import, storage backends, Apple Notes support) is that we now have **production benchmark data proving the search quality thesis**.

The README currently explains the architecture well but doesn't have concrete evidence for WHY hybrid search matters. We now have that evidence.

## The Benchmark

We ran 12 queries across 4 difficulty tiers against a production brain with 13,106 indexed pages and 19,979 chunks. Three methods compared:

**Method 1: `grep -ril` (filesystem search)**
- Average: 231ms
- Correct #1 result: 0 out of 12

**Method 2: `gbrain search` (keyword: pg_trgm + tsvector)**
- Average: 666ms
- Correct #1 result: 8 out of 12

**Method 3: `gbrain query` (hybrid: keyword + pgvector semantic + RRF)**
- Average: 2,434ms
- Correct #1 result: 12 out of 12

### Raw results (sanitized — no real names)

```
Query                                    grep       search     query(semantic)
─────────────────────────────────────────────────────────────────────────────
TIER 1: Entity lookup (known names)
  "John Smith"                           188ms ❌    635ms ✅    1801ms ✅
  "Acme Corp CEO"                        198ms ❌    595ms ✅    1497ms ✅
  "Project Alpha"                        190ms ❌    665ms ✅    2010ms ✅

TIER 2: Topic/concept recall
  "founder mode"                         186ms ❌    671ms ✅    1714ms ✅
  "Series A deal terms"                  256ms ❌    662ms ❌    2650ms ✅
  "batch selection criteria"             198ms ❌    690ms ✅    2617ms ✅

TIER 3: Semantic (no exact keyword match)
  "founders building developer tools"    237ms ❌    696ms ⚠️    2719ms ✅
  "shame as fuel for ambition"           384ms ❌    711ms ✅    2610ms ✅
  "what makes a 10x company"             194ms ❌    724ms ⚠️    2904ms ✅

TIER 4: Cross-domain / relational
  "people who know both X and Y"         198ms ❌    576ms ❌    3181ms ✅
  "restaurants near resort"              366ms ❌    689ms ✅    2836ms ✅
  "original ideas about abundance"       188ms ❌    682ms ⚠️    2680ms ✅
```

### What grep actually returned (this is the damning part)

- For "John Smith" → returned a project README that mentions the name in passing, not the actual person's dossier
- For "Acme Corp CEO" → returned an index file, not the person or company page
- For "Series A deal terms" → returned an event page about the Olympics (!)
- For "what makes a 10x company" → returned a demo day page (5 different queries returned this same page)
- For "restaurants near resort" → returned an adversary tracking page (!!)
- For "original ideas about abundance" → returned an inbox README

**grep returned `yc-demo-day` as the top result for 5 completely different queries.** It's matching incidental word occurrences, not answering the question.

### The core insight (PUT THIS IN THE README)

This is NOT a speed story. grep is 10x faster. That's irrelevant.

This is a **correctness story**. At 13,000+ files, grep returns noise. It finds files that *contain a word from your query*, not files that *answer your question*. When your brain has 3,000 people pages, 5,800 archived notes, and 500+ media pages, the word "founder" appears in hundreds of files. grep can't tell which one you want. It returns whichever file the filesystem scanner hits first.

The practical consequence: **grep-based lookup causes the agent to hallucinate.** Not because the LLM is making things up — because it's being fed the wrong context. You ask "who is John Smith?" and the agent gets a project README instead of the person's dossier. Now it's generating a response from irrelevant context. The hallucination isn't in the model — it's in the retrieval.

Hybrid search eliminates this. The semantic layer understands that "shame as fuel for ambition" should find your essay about founder psychology, not a file that happens to contain the word "shame." The keyword layer ensures exact names still match instantly. RRF fusion combines both signals.

**The 2 seconds of extra latency buys you the right answer.** In a system where the agent is already spending 5-10 seconds thinking before it responds, 2 seconds of retrieval is invisible. But feeding the agent wrong context is catastrophic — it poisons the entire response.

## What to change in the README

1. **Add a "Why Not Just Grep?" section** (or expand the existing "Why this exists" section) with the benchmark data. This should be near the top — it's the strongest argument for why gbrain exists. Use the sanitized benchmark numbers, not real names.

2. **Add the hallucination argument.** The key framing: grep doesn't cause grep to hallucinate — grep causes the *agent* to hallucinate by feeding it wrong context. This is a concrete, measurable problem, not a theoretical one.

3. **Add a "Benchmark" section** with instructions for running the benchmark yourself: `bash skills/benchmark-gbrain/scripts/benchmark.sh`. Users can verify on their own data.

4. **Update the "Why this exists" narrative.** The current version mentions grep falling apart at scale but doesn't have concrete numbers. Now we have them. The story should be: "At 500 files grep works. At 13,000 files, grep returned the correct top result 0 out of 12 times. Here's what it returned instead."

5. **Update the v0.4.0 section in the changelog** if it exists, or add release notes mentioning the benchmark skill.

6. **Keep the tone.** The README's voice is good — direct, technical, opinionated, no marketing fluff. The benchmark section should match: show the data, explain what it means, don't oversell.

## What NOT to change

- Don't touch the architecture diagrams, they're good
- Don't change the install/setup flow
- Don't remove the "What one brain looks like" section
- Don't change the knowledge model explanation
- Don't add fluff or marketing language
- **IMPORTANT: Do not reference any real people by name in the benchmark section.** Use generic examples ("a person page", "a company page", "an essay about founder psychology"). The benchmark queries in the skill use real names but the README should not.

## Also scrub existing real-name references

The README currently has these real-name references that should be genericized:
- Line 24: "Pedro and Diana" → use generic names
- Line 172: "Pedro", "Brex" → use generic examples
- Lines 12, 34: "dossiers" is fine (generic term), keep it

Replace with plausible but clearly fictional examples. Don't use "Alice and Bob" — too cliché. Use something like "Jordan" and "Sarah" or similar.

## Files to edit

- `README.md` — main changes + scrub real names from examples
- `CHANGELOG.md` — add benchmark skill to v0.4.0 section if not already there

## How to run this

```bash
cd /tmp/gbrain-product
# Read the current README
cat README.md
# Read the benchmark skill for reference
cat skills/benchmark-gbrain/SKILL.md 2>/dev/null || cat /data/.openclaw/workspace/skills/benchmark-gbrain/SKILL.md
# Read the benchmark script for the actual test methodology
cat skills/benchmark-gbrain/scripts/benchmark.sh 2>/dev/null || cat /data/.openclaw/workspace/skills/benchmark-gbrain/scripts/benchmark.sh
# Make edits to README.md
# Verify nothing references real people in the benchmark section
grep -n "Pedro\|Benioff\|Legion\|Garry\|adversary\|oppo" README.md
```
