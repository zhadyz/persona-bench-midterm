# Persona-Bench: Reproducibility Guide

**CSC669/899 — AI Safety Benchmark "Social World"**
**Steps 1-3: Seed Data → Interview → Expert Verification → Social Circles**

This guide enables a total stranger to reproduce our pipeline from scratch using Claude.

---

## Prerequisites

- **Claude**: Claude Opus 4.6 (1M context) via Claude Code CLI or claude.ai
- **Python 3.10+** with the `datasets` library (`pip install datasets`)
- **Internet access** to download the NVIDIA dataset from HuggingFace

---

## Step 0: Setup

### 0.1 Create the folder structure

```
mkdir -p data/step1_seed data/step2_interview data/step2b_verification data/step3_social_circles prompts
```

### 0.2 Get the interview questions

The full 109-question interview protocol is from **Park et al. 2024**, "Generative Agent Simulations of 1,000 People" (arXiv:2411.10109), Table 7 (pages 60-64). We have extracted these into `data/interview_questions_table7.json`.

If reproducing from scratch, the questions can be extracted from the paper's PDF or copied from our provided JSON file.

---

## Step 1: Pull Seed Data from NVIDIA

### Source
**NVIDIA Nemotron-Personas-USA** — a dataset of 6 million synthetic personas built on a Probabilistic Graphical Model (PGM) trained on US Census Bureau ACS data.

- HuggingFace: https://huggingface.co/datasets/nvidia/Nemotron-Personas-USA
- License: CC BY 4.0

### How to pull the first 5 personas

```python
from datasets import load_dataset
import json

ds = load_dataset("nvidia/Nemotron-Personas-USA", split="train")

for i in range(5):
    persona = dict(ds[i])
    # Extract name from persona field
    import re
    name_match = re.match(r'^([A-Za-z]+ [A-Za-z]+)', persona.get('persona', ''))
    name = name_match.group(1) if name_match else f'persona_{i+1}'
    safe_name = name.lower().replace(' ', '_')

    with open(f'data/step1_seed/{safe_name}_seed.json', 'w', encoding='utf-8') as f:
        json.dump(persona, f, indent=2, ensure_ascii=False)
    print(f"Saved: {safe_name}_seed.json")
```

### Expected output
5 JSON files in `data/step1_seed/`, each containing all NVIDIA narrative fields:
- `uuid`, `professional_persona`, `sports_persona`, `arts_persona`, `travel_persona`, `culinary_persona`, `persona`, `cultural_background`, `skills_and_expertise`, `hobbies_and_interests`, `career_goals_and_ambitions`, `sex`, `age`, `marital_status`, `education_level`, `occupation`, `city`, `state`, `zipcode`, `country`

### The 5 personas we used

| # | Name | Age | Sex | Location | Education | Occupation |
|---|------|-----|-----|----------|-----------|------------|
| 1 | Mary Alberti | 28 | F | Madison, WI | High school | Fast food worker |
| 2 | Alicia Gonzalez | 29 | F | Chico, CA | Bachelor's | CS researcher |
| 3 | Deeva Cintron | 85 | F | Saline, MI | Some college | Retired |
| 4 | Maria Buendia | 34 | F | Lilburn, GA | 9th-12th no diploma | No occupation |
| 5 | Julio Simmons | 49 | M | Rochester, NY | Graduate | Not in workforce |

### Verification
- Each file should contain all 20+ NVIDIA fields
- Demographics should be plausible (age matches life stage, location matches cultural background)

---

## Step 2: Run the Full Interview

### Source
**American Voices Project** interview protocol, adapted from Park et al. 2024 (Table 7).
- 109 questions across 17 blocks
- Covers: life story, family, neighborhood, household, daily routine, work, law enforcement, politics, race, health, mental health, substances, religion, social media, finances, occupation details, demographics, childhood, hopes/values
- Total interview time: ~7,200 seconds (~2 hours)

### How to run

**For each persona**, open a new Claude conversation and paste the full prompt from `prompts/step2_interview_full.txt`, replacing all `{{placeholders}}` with the actual values from that persona's seed JSON file.

**Placeholders to replace:**
```
{{participant_id}}      → the persona's uuid from seed
{{name}}                → extracted name (e.g., "Mary Alberti")
{{age}}                 → age (e.g., 28)
{{sex}}                 → sex (e.g., "Female")
{{marital_status}}      → marital_status field
{{education_level}}     → education_level field
{{occupation}}          → occupation field
{{city}}                → city field
{{state}}               → state field
{{zipcode}}             → zipcode field
{{country}}             → country field
{{professional_persona}} → full professional_persona text
{{sports_persona}}      → full sports_persona text
{{arts_persona}}        → full arts_persona text
{{travel_persona}}      → full travel_persona text
{{culinary_persona}}    → full culinary_persona text
{{persona}}             → full persona text
{{cultural_background}} → full cultural_background text
{{skills_and_expertise}} → full skills_and_expertise text
{{hobbies_and_interests}} → full hobbies_and_interests text
{{career_goals_and_ambitions}} → full career_goals_and_ambitions text
```

### Critical rules enforced by the prompt
1. **Behaviors, never labels** — for health, religion, politics, mental health, finances, and substances, the persona describes BEHAVIORS only (e.g., "I check my levels three times a day" instead of "I have diabetes")
2. **Voice consistency** — answers match the persona's age, education, occupation, and cultural background
3. **Follow-ups** — 2-3 follow-up questions for narrative questions (>25s time limit), 0 for short factual ones

### Expected output
One JSON file per persona in `data/step2_interview/`, containing:
- `interview_metadata` — participant info, date, protocol, model version
- `transcript` — all 109 questions with responses and follow-ups
- `extracted_profile` — the new schema:
  - Demographics header (participant_id, name, age, gender, marital_status, location, education, occupation)
  - `preferences` — 15 broad life domains (health, food, travel, social, home, entertainment, hobbies, work, arts, sports, nature, technology, community, music, fashion)
  - `summaries` — free-text paragraphs from 4 expert perspectives (demographer, behavioral_economist, political_scientist, psychologist)
  - `others_misc` — 6 typed subcategories with explicit inclusion criteria (catch-all for info not in preferences or summaries: cultural/linguistic, life events, sensory/physical, media diet, tech/tools, unresolved)
  - `hidden_facts` — internal answer key for Steps 4-7: ground-truth facts (health, religion, politics, etc.) with embedding strategies for distributing behavioral evidence across app logs. Never shown to the evaluator LLM.

### Verification
- Valid JSON
- All 109 question IDs present (Q01-Q109)
- 60-90 follow-up exchanges total
- No clinical labels, party names, or religious labels in persona responses
- All 15 preference categories present (some may have empty details arrays)
- All 4 expert summaries present and non-empty

---

## Step 2b: Expert Verification (4-Direction)

### Source
4-expert panel adapted from Park et al. 2024. Each expert has a broad disciplinary mandate.

### Expert definitions

**Demographer**: demographic traits, social status, age, gender, ethnicity, geography, education, occupation, household/life-course cues

**Behavioral Economist**: motivations, incentives, uncertainty, peer influence, social expectations, habits, identity, loss aversion, risk, time preferences, default effects — assessed across health/lifestyle, productivity/work, personal finance/savings, relationships/social life, public policy/community behavior

**Political Scientist**: civic orientation, institutional trust, community attachment, response to authority and collective action

**Psychologist**: personality/trait expression, emotional regulation and coping, self-concept and identity, cognitive style, relational dynamics, motivational architecture. Outputs behavioral patterns only — no clinical diagnoses or DSM labels.

### How to run

**For each persona**, open a new Claude conversation and paste the full prompt from `prompts/step2b_expert_verification.txt`, replacing:
- `{{INSERT_FULL_SEED_JSON_HERE}}` → paste the entire contents of that persona's seed JSON
- `{{INSERT_TRANSCRIPT_JSON_HERE}}` → paste the `transcript` field from that persona's Step 2 interview output
- `{{INSERT_EXTRACTED_PROFILE_JSON_HERE}}` → paste the `extracted_profile` field from that persona's Step 2 interview output
- `{{participant_id}}` and `{{name}}` → from the seed data

### 4-direction verification
The verification checks ALL of these:
1. **Seed ↔ Interview Q&A** — does the interview contradict the NVIDIA seed data?
2. **Interview Q&A ↔ Extracted Profile** — does the extracted_profile faithfully represent the interview?
3. **Extracted Profile ↔ Itself** — are there self-contradictions within summaries, preferences, or others_misc?
4. **Completeness** — is anything from the interview missing from the entire extracted_profile?

Each expert produces observations for all 4 directions, plus a free-text summary.

### Expected output
One JSON file per persona in `data/step2b_verification/`, containing:
- 4 expert sections, each with 4 direction sub-sections (50-70 total observations)
- Overall verdict: PASS, PASS_WITH_REVISIONS, or FAIL
- Inconsistency resolutions (if any)
- Gap dispositions (CARRY_FORWARD, DROP, or NEEDS_ENRICHMENT)
- 4 expert summaries (3-5 sentences each with Q## citations)

### Verification
- Valid JSON
- All 4 experts present, each with all 4 directions
- Each observation has verdict and source references
- Expert summaries are non-empty free text
- Zero or few inconsistencies (NVIDIA seed data is well-formed)

---

## Step 3: Social Circle Generation

### Source
**PersonaHub Persona-to-Persona method** (Ge et al., 2024, Tencent AI Lab)

### How to run

**For each persona**, open a new Claude conversation and paste the full prompt from `prompts/step3_social_circles.txt`, replacing:
- `{{INSERT_EXTRACTED_PROFILE_JSON_HERE}}` → the `extracted_profile` section from Step 2 output
- `{{INSERT_EXPERT_SUMMARIES_JSON_HERE}}` → the `expert_summaries` section from Step 2b output
- `{{INSERT_TRANSCRIPT_JSON_HERE}}` → the `transcript` section from Step 2 output
- `{{participant_id}}` and `{{name}}` → from the seed data

### Constraints
- 5 closest people to the persona
- At least 1 family member
- At least 1 friend outside work
- At least 1 work colleague or community contact (adjusted for retired/unemployed personas)
- Diverse ages, genders, and backgrounds
- Every person must cite Q## evidence from the interview

### Expected output
One JSON file per persona in `data/step3_social_circles/`, containing:
- `social_circle_metadata` — participant info, date, model, method
- `social_circle` — array of 5 people, each with:
  - Demographics (name, age, gender, occupation, location, education)
  - Background (how they met, years known, why close)
  - Personality mini-profile (traits, interests, communication style)
  - Shared activities
  - Communication frequency
  - Recent text topics (past 2-3 months)
  - Evidence from interview (Q## references)

### Verification
- Valid JSON
- Exactly 5 social circle members
- At least 1 family, 1 friend, 1 work/community contact
- All members have Q## evidence citations
- Demographics are plausible for the persona's location and life stage

---

## Step 3b: Social Circle Verification

### Source
Custom 2-direction verification prompt checking social circles against both the interview and extracted profile.

### How to run

**For each persona**, open a new Claude conversation and paste the full prompt from `prompts/step3_social_circles_verification.txt`, replacing:
- `{{INSERT_TRANSCRIPT_JSON_HERE}}` → the `transcript` field from Step 2 output
- `{{INSERT_EXTRACTED_PROFILE_JSON_HERE}}` → the `extracted_profile` field from Step 2 output
- `{{INSERT_SOCIAL_CIRCLES_JSON_HERE}}` → the entire Step 3 social circles output
- `{{participant_id}}` and `{{name}}` → from the seed data

### 2-direction verification
1. **Social Circles ↔ Interview Q&A** — do generated people contradict interview evidence?
2. **Social Circles ↔ Extracted Profile** — are shared activities and topics consistent with stated preferences?

Plus structural checks (family/friend/colleague diversity, evidence citations).

### Expected output
One JSON file per persona in `data/step3_social_circles/`, named `{name}_social_circles_verification.json`, containing:
- Per-person verification with observations for both directions
- Structural checks (family, friend, colleague, diversity, evidence)
- Overall verdict: PASS, PASS_WITH_REVISIONS, or FAIL
- Lists of inconsistencies and gaps found

### Verification
- Valid JSON
- All 5 social circle members verified
- Each has observations for both directions
- Structural checks section present

---

## Summary of Files Produced

```
data/
  interview_questions_table7.json          # 109 questions from Park et al. Table 7
  step1_seed/
    {name}_seed.json                       # NVIDIA seed data (5 files)
  step2_interview/
    {name}_interview.json                  # Full 109-Q interview + extracted_profile (5 files)
  step2b_verification/
    {name}_verification.json               # 4-direction expert verification (5 files)
  step3_social_circles/
    {name}_social_circles.json             # 5 closest people (5 files)
    {name}_social_circles_verification.json # Social circle verification (5 files)

prompts/
  step2_interview.txt                      # Abbreviated interview prompt (12 Q, for reference)
  step2_interview_full.txt                 # Self-contained interview prompt (109 Q)
  step2b_expert_verification.txt           # 4-direction verification prompt
  step3_social_circles.txt                 # Social circles generation prompt
  step3_social_circles_verification.txt    # Social circles verification prompt
  step4_messenger_snippets.txt             # Messenger log prompt (Steps 4-7, for future use)
  step4_messenger_verification.txt         # Messenger verification prompt (future)
  step5_calendar.txt                       # Calendar log prompt (future)
  step5_calendar_verification.txt          # Calendar verification prompt (future)
  step6_evaluation.txt                     # Evaluation prompt (future)
  step7_test_cases.txt                     # Test case generation prompt (future)
```

**Total: 26 data files + 11 prompt templates (4 active for Steps 1-3b, 7 for future Steps 4-7)**

---

## References

| Paper | Used In | Citation |
|-------|---------|----------|
| Generative Agent Simulations of 1,000 People | Steps 2, 2b | Park et al. 2024, arXiv:2411.10109 |
| PersonaHub: An Easy-to-Use and Large-Scale Persona-Driven Data Synthesis Tool | Step 3 | Ge et al. 2024, Tencent AI Lab |
| NVIDIA Nemotron-Personas-USA | Step 1 | NVIDIA, HuggingFace CC BY 4.0 |
| PersonaMem-v2: Towards Personalized Intelligence via Learning Implicit User Personas and Agentic Memory | Design Principles | Jiang et al. 2025, arXiv:2512.06688 |

## Tools Used

- **Model**: Claude Opus 4.6 (1M context) by Anthropic
- **Interface**: Claude Code CLI (no API calls — all prompts executed manually)
- **Dataset**: HuggingFace `datasets` library for NVIDIA data access

---

## License

Apache 2.0
