# Persona-Bench

**CSC669/899 — AI Safety Benchmark "Social World"**

## What This Is

A reproducible pipeline that takes seed personas from the NVIDIA Nemotron-Personas-USA dataset and produces richly detailed persona profiles with social circles, verified by a 4-expert panel. The pipeline is designed so that a total stranger can reproduce every step using Claude.

## How It Was Built

The entire pipeline was executed using **Claude Opus 4.6 (1M context)** via Claude Code. Each step takes a self-contained prompt template from `prompts/` and produces structured JSON output. No external libraries, APIs, or training were required beyond the HuggingFace `datasets` library for seed data access.

We ran the pipeline on the **first 5 personas** from the NVIDIA dataset:

| # | Name | Age | Location | Education | Occupation |
|---|------|-----|----------|-----------|------------|
| 1 | Mary Alberti | 28 | Madison, WI | High school | Fast food worker |
| 2 | Alicia Gonzalez | 29 | Chico, CA | Bachelor's | CS researcher |
| 3 | Deeva Cintron | 85 | Saline, MI | Some college | Retired |
| 4 | Maria Buendia | 34 | Lilburn, GA | 9th-12th no diploma | No occupation |
| 5 | Julio Simmons | 49 | Rochester, NY | Graduate | Not in workforce |

## Pipeline

```
Step 1: Seed Data (NVIDIA Nemotron-Personas-USA, HuggingFace)
  ↓
Step 2: Full 109-Question Interview (Park et al. 2024, Table 7)
  ↓       → interview transcript + extracted_profile
Step 2b: 4-Expert Verification (Demographer, Behavioral Economist,
  ↓       Political Scientist, Psychologist)
  ↓       → verification observations + expert summaries
Step 3: Social Circles (PersonaHub Persona-to-Persona, Ge et al. 2024)
          → 5 closest people per persona
```

## Repository Structure

```
prompts/                              # Self-contained prompt templates
  step2_interview_full.txt            # 109-question interview (generic)
  step2b_expert_verification.txt      # 4-expert verification (generic)
  step3_social_circles.txt            # Social circle generation (generic)

data/                                 # All pipeline outputs
  interview_questions_table7.json     # 109 questions extracted from Table 7
  step1_seed/                         # NVIDIA seed data (5 personas)
  step2_interview/                    # Interview transcripts + extracted profiles
  step2b_verification/                # Expert verification + summaries
  step3_social_circles/               # Social circles (5 people per persona)

REPRODUCIBILITY_GUIDE.md              # Step-by-step guide for reproduction
PIPELINE_DESIGN.md                    # Full pipeline design document
REFERENCES.md                         # Paper references and team credits
```

## Key Design Principles

- **Behaviors, never labels**: Sensitive topics (health, religion, politics, mental health) are expressed through observable behaviors, never clinical or categorical labels. This enables the reverse pass — an AI evaluator must infer labels from behavioral clues alone.
- **Self-contained prompts**: Every prompt includes all context needed. No assumed prior state.
- **Generic placeholders**: Prompts use `{{field}}` placeholders so they work for any NVIDIA persona.
- **PersonaMem-v2 alignment** (arXiv:2512.06688): Implicit preference revelation, broad preference domains, compact summaries, multi-step validation.
- **Broad expert definitions**: Each expert applies their full disciplinary lens, not a narrow checklist.

## Extracted Profile Schema

The interview produces a structured profile with:
- **Demographics**: participant_id, name, age, gender, marital_status, location, education, occupation
- **Preferences**: 15 broad life domains (health, food, travel, social, home, entertainment, hobbies, work, arts, sports, nature, technology, community, music, fashion)
- **Summaries**: Free-text paragraphs from 4 expert perspectives
- **Others/Misc**: 6 typed subcategories for information not captured elsewhere

## References

| Paper | Used In |
|-------|---------|
| Generative Agent Simulations of 1,000 People (Park et al. 2024) | Steps 2, 2b |
| PersonaHub (Ge et al. 2024, Tencent AI Lab) | Step 3 |
| NVIDIA Nemotron-Personas-USA | Step 1 |
| PersonaMem-v2 (Jiang et al. 2025) | Design principles |

## Team

- **Abdul** — Pipeline implementation, prompt engineering
- **Miles** — Persona-to-Persona approach (Step 3)
- **Professor Isabel** — Pipeline design, step definitions, references

## License

Apache 2.0
