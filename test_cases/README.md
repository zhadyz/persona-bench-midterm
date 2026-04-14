# Step 7: Test Case Generation & Verification Design

## Repository

**GitHub**: [zhadyz/persona-bench-midterm](https://github.com/zhadyz/persona-bench-midterm)

All test cases are in the `test_cases/` directory. Reproducible prompts are in `prompts/`.

## What We Did

Generated 350 benchmark test cases (70 per persona, 5 personas) that evaluate whether an LLM can reconstruct a persona's identity from reading only their app logs (messenger + calendar).

**Inputs used:**
- Corrected extracted profiles from Step 2b verification (ground truth / answer key)
- Dynamic events from Anmol's app log files (6 events per persona)

**LLM used for test case generation**: Claude Opus 4.6 (1M context)

**LLM used for benchmark evaluation**: Claude Opus 4.6 (1M context)

## Test Case Distribution (per persona)

| Type | Count | Difficulty | What It Tests |
|------|-------|-----------|---------------|
| Type 1: Simple Fact-Check | 5 | Easy | Single-source retrieval |
| Type 2: Cross-Log Fact-Check | 8 | Medium | Connect 2+ sources (messenger + calendar) |
| Type 3: Dynamic Preference Tracking | 5 | Medium-Hard | Detect change over time |
| Type 4: Reasoning | 52 | Hard | Multi-source inference across full log corpus |
| **Total** | **70** | | |

## Files

| File | Persona | Test Cases |
|------|---------|-----------|
| `julio_simmons_test_cases.json` | Julio Simmons | 70 |
| `mary_alberti_test_cases.json` | Mary Alberti | 70 |
| `alicia_gonzalez_test_cases.json` | Alicia Gonzalez | 70 |
| `deeva_cintron_test_cases.json` | Deeva Cintron | 70 |
| `maria_buendia_test_cases.json` | Maria Buendia | 70 |
| `julio_benchmark_results.json` | Julio Simmons | 15-case evaluation |

## Current Accuracy

Preliminary benchmark run on Julio Simmons (15 sampled test cases):

| Type | Cases | Score |
|------|-------|-------|
| Type 1: Simple Fact-Check | 3 | 50% |
| Type 2: Cross-Log | 2 | 100% |
| Type 3: Dynamic Tracking | 2 | 50% |
| Type 4: Reasoning | 8 | 56% |
| **Overall** | **15** | **60%** |

Target accuracy is 40-50%. Current 60% is above target because the app logs lack concrete specifics (zero dollar amounts, few food/health details), causing most answers to score PARTIAL rather than CORRECT or INCORRECT. Once app logs are revised with more concrete behavioral details, accuracy should settle into the target range.

## How to Run the Benchmark

The test cases are ready to feed into OpenClaw or any evaluation framework:

1. **Input to the LLM**: raw app logs only (messenger conversations + calendar events). Strip all metadata, theme plans, and cross-app indices.
2. **Ask**: the `question` field from each test case.
3. **Score against**: the `ground_truth` field (CORRECT / PARTIAL / INCORRECT).
4. **Fresh session per persona** — no cross-session context contamination.

Reproducible prompts:
- `prompts/step7_test_cases_v2.txt` — how to generate test cases for new personas
- `prompts/step7_benchmark_runner.txt` — how to run and score the benchmark

---

## Verification Design (Inspired by SWE-bench Verified)

### Background: What OpenAI Did with SWE-bench

SWE-bench is a coding benchmark with 2,294 tasks from real GitHub issues. OpenAI discovered the original had quality problems, so they created **SWE-bench Verified**:

- **93 software developers** manually reviewed 1,699 samples
- **3 annotators per sample** independently evaluated each one
- **Two criteria checked per sample:**
  1. Is the problem statement clear enough to solve? (underspecified = unfair)
  2. Do the tests actually accept correct solutions? (flawed tests = false failures)
- **Severity scale 0-3** per criterion (0-1 = minor, 2-3 = severe/remove)
- **Result: 68.3% of samples were rejected** as inadequate
- Final verified set: **500 out of 1,699** (29.4% survival rate)
- Later audit found even the verified set had issues: 59.4% of hard-to-solve problems had flawed test cases

### What We Can Borrow for Persona-Bench

The same pattern applies to our benchmark. We have two things to verify:

**1. App Log Verification (Anmol's output)**

Are the hidden facts actually inferable from the logs? For each preference/theme:

| Criterion | Severity Scale | What It Checks |
|-----------|---------------|----------------|
| Evidence exists | 0-3 | Are there concrete details in the logs that support this fact? |
| Evidence is implicit | 0-3 | Is the fact shown through behavior, not stated directly? |
| Evidence is distributed | 0-3 | Does it appear across 2+ sources (chat + calendar)? |
| No contradiction | 0-3 | Do the logs contradict the extracted profile anywhere? |

Automated approach: For each preference in the extracted profile, search the raw logs for related keywords/details. Flag any preference with zero matches as "evidence missing." LLM-based pass: ask an LLM to read the logs and try to infer each fact — if it can't, the fact isn't sufficiently embedded.

**2. Test Case Verification (Abdul's output)**

Are the test cases fair and answerable? For each test case:

| Criterion | Severity Scale | What It Checks |
|-----------|---------------|----------------|
| Answerable from logs | 0-3 | Can this question be answered from app logs alone? |
| Ground truth is correct | 0-3 | Does the answer match the verified extracted profile? |
| Not a duplicate | 0-3 | Does this require a different reasoning path than other cases? |
| Difficulty is calibrated | 0-3 | Is the type assignment (1-4) appropriate for the actual difficulty? |

Automated approach: Run each test case against the logs using an LLM. If Type 1 cases score INCORRECT, either the logs are broken or the test case is unfair. If Type 4 cases score CORRECT consistently, they're too easy.

### Proposed Verification Pipeline

```
For each persona:
  1. AUTOMATED: Search logs for keyword matches against profile preferences
     → Produce coverage report (X of Y preferences have evidence)

  2. AUTOMATED: Run LLM on raw logs + test case questions
     → Score each answer against ground truth
     → Flag: test cases that score 0 (evidence might be missing)
     → Flag: test cases where Type 1 scores INCORRECT (unfair or broken)

  3. MANUAL (1 persona only, e.g., Alicia): Human reviews flagged cases
     → Confirm whether failure is log gap vs. LLM limitation
     → Calibrate the automated scoring

  4. AUTOMATED (remaining personas): Apply calibrated criteria
     → Accept/reject test cases based on step 3 thresholds
     → Produce "Persona-Bench Verified" subset
```

This scales: manual verification on 1 persona calibrates the automated process for the other 99.

### Key Lesson from SWE-bench Verified

OpenAI's experience shows that **even verified benchmarks degrade over time**. Their later audit found 59.4% of "hard" problems had flawed tests. For Persona-Bench:

- Version the test cases (V1, V2, etc.)
- Re-run verification after any app log revision
- Track which test cases consistently produce PARTIAL scores — those likely have evidence gaps
- Maintain the "reverse inferability" principle: only test cases where the evidence actually exists in the logs should count toward the benchmark score

Sources:
- [Introducing SWE-bench Verified | OpenAI](https://openai.com/index/introducing-swe-bench-verified/)
- [Why SWE-bench Verified no longer measures frontier coding capabilities | OpenAI](https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/)
- [SWE-bench Verified | swebench.com](https://www.swebench.com/verified.html)
- [SWE-bench Annotation Instructions (PDF)](https://cdn.openai.com/introducing-swe-bench-verified/swe-b-annotation-instructions.pdf)
- [SWE-bench: A Comprehensive Review | atoms.dev](https://atoms.dev/insights/swe-bench-a-comprehensive-review-of-its-fundamentals-methodology-impact-and-future-directions/6c3cb9820d3b44e69862f7b064c1fd1e)
