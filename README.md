# Persona-Bench Midterm

**CSC669/899 — AI Safety Benchmark "Social World"**

## What This Is

A 7-step pipeline that takes a single seed persona (Mary Alberti) from the NVIDIA Nemotron-Personas-USA dataset and produces realistic app logs (messenger + calendar) with embedded hidden facts, then evaluates whether an AI agent can reconstruct the persona from those logs alone.

## How It Was Built

The entire pipeline was executed using **Claude** (Anthropic). Each step takes a prompt template from `prompts/` and the output from the previous step as input. No external libraries, APIs, or training were required.

```
Step 1: Seed Data (NVIDIA)
  ↓ input to
Step 2: LLM Interview → interview_output.json
  ↓ input to
Step 2b: 4-Expert Verification → verification_output.json
  ↓ input to
Step 3: Social Circles (Persona-to-Persona) → social_circle_output.json
  ↓ input to
Step 4: Messenger Logs + Needle Insertion → messenger_output.json
Step 5: Calendar Logs → calendar_output.json
  ↓ input to
Step 6: Coverage Evaluation → evaluation_output.json
Step 7: 75 Test Cases → test_cases_output.json
```

## Repository Structure

```
prompts/                          # All prompt templates (the "source code")
  step2_interview.txt             # Stanford interview generation
  step2b_expert_verification.txt  # 4-expert verification
  step3_social_circles.txt        # Persona-to-Persona generation
  step4_messenger_snippets.txt    # Messenger with needle insertion
  step4_messenger_verification.txt
  step5_calendar.txt              # Calendar generation
  step5_calendar_verification.txt
  step6_evaluation.txt            # Coverage evaluation
  step7_test_cases.txt            # Test case generation

step1_seed_data/                  # NVIDIA seed data for Mary Alberti
step2_interview/                  # Interview transcript + extracted profile
step2b_expert_verification/       # 4-expert verification + preferences + inconsistency resolution
step3_social_circles/             # 5 closest people + text-to-persona method
step4_messenger_logs/             # 19 snippets across 5 conversations + verification
step5_calendar_logs/              # 47 calendar entries + verification
step6_evaluate/                   # Coverage scores + spec compliance
step7_test_cases/                 # 75 evaluation test cases with evidence

PIPELINE_DESIGN.md                # Full pipeline design document
REFERENCES.md                     # Paper references and team credits
```

## Pipeline Steps

| Step | Input | Prompt | Output |
|------|-------|--------|--------|
| 1 | NVIDIA Nemotron-Personas-USA | — | `mary_alberti_seed.json` |
| 2 | Seed data | `step2_interview.txt` | `interview_output.json` |
| 2b | Seed + interview | `step2b_expert_verification.txt` | `verification_output.json` |
| 3 | Interview output | `step3_social_circles.txt` | `social_circle_output.json` |
| 4 | Interview + circles | `step4_messenger_snippets.txt` | `messenger_output.json` |
| 5 | Interview + circles | `step5_calendar.txt` | `calendar_output.json` |
| 6 | Messenger + calendar | `step6_evaluation.txt` | `evaluation_output.json` |
| 7 | Messenger + calendar | `step7_test_cases.txt` | `test_cases_output.json` |

## Key Methods

- **Seed Data**: NVIDIA Nemotron-Personas-USA (Census-aligned PGM, 6M personas)
- **Interview**: Stanford protocol from Park et al. 2024 (Table 7, 12 questions, ~30 min)
- **Verification**: 4 expert agents — Psychologist, Behavioral Economist, Political Scientist, Demographer
- **Social Circles**: PersonaHub Persona-to-Persona method (Tencent AI Lab, 2024)
- **Needle Insertion**: Sequential-NIAH — 3 types: synthetic temporal, real temporal, real logical
- **Both PersonaHub methods used**: Text-to-Persona (Steps 1→2) and Persona-to-Persona (Steps 2→3)

## Test Cases (Step 7)

75 evaluation test cases across 4 types:

| Type | Count | Description |
|------|-------|-------------|
| Type 1: Simple Fact-Check | 1 | Single-source retrieval |
| Type 2: Cross-Log Fact-Check | 3 | Cross-reference messenger + calendar |
| Type 3: Dynamic Preference Tracking | 1 | Detect behavioral change over time |
| Type 4: Reasoning | 70 | Multi-source reasoning across apps and conversations |

## References

| Paper | Used In |
|-------|---------|
| Generative Agent Simulations of 1,000 People (Park et al. 2024) | Steps 2, 2b |
| PersonaHub (Tencent AI Lab, 2024) | Step 3 |
| NVIDIA Nemotron-Personas-USA | Step 1 |
| Crafting Customisable Characters with LLMs (SimsChat) | Step 4 |
| The Steerability of LLMs Toward Data-Driven Personas | Step 4 |
| PICLe: Persona In-Context Learning | Step 4 |
| Solo Performance Prompting (SPP) | Step 4 |
| Sequential-NIAH | Step 4 |

## License

Apache 2.0
