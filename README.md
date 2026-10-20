# Intern Interview Question Generator

Generate role-specific technical and behavioral interview questions for interns, tailored to a job description and a candidate's background.

The tool uses **retrieval-augmented generation (RAG)**. It finds the most relevant questions in your own question bank, uses them as style and difficulty references, and asks an LLM to write a fresh question set. The output is then checked automatically before a human reviews it.

## Features

- **Role-specific output:** questions are tied to skills extracted from the job description
- **Candidate-aware:** includes project-based questions drawn from the intern's own background
- **Privacy-first:** strips emails, phone numbers, URLs, and identity fields before any text reaches the model
- **Grounded in your bank:** retrieved questions keep the style and difficulty consistent with your team's standards
- **Automatic validation:** schema, duplicates, skill coverage, and a protected-topic filter, with automatic retries
- **Model-agnostic:** works with the Anthropic API out of the box, or a local LLaMA model
- **Human in the loop:** every set is marked `ready_for_review` or `needs_human_attention`

## How it works

```
Job description ─┐
                 ├─> strip PII ─> extract skills ─> retrieve similar bank questions
Intern profile ──┘                                              │
                                                                v
Question bank ──────────────────────────────────────────> build prompt
                                                                │
                                                                v
                                                         LLM generation
                                                                │
                                                                v
                                              validate ──fail──> retry with feedback
                                                  │ pass
                                                  v
                                       interview_set.json (for human review)
```

## Project structure

```
intern_question_generator/
├── generator.py                 # full pipeline and CLI
├── question_bank.sample.json    # example tagged question bank
├── requirements.txt
└── README.md
```

## Installation

Requires Python 3.10 or newer.

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
export ANTHROPIC_API_KEY=your_key_here
```

The first run downloads the embedding model (`all-MiniLM-L6-v2`), so it needs internet access once.

## Quick start

1. Save a job description to `jd.txt` and an intern profile to `profile.txt`.
2. Preview the prompt without calling any model:

   ```bash
   python generator.py --role "Data Analyst Intern" \
     --jd jd.txt --profile profile.txt \
     --bank question_bank.sample.json --dry-run
   ```

3. Generate the full question set:

   ```bash
   python generator.py --role "Data Analyst Intern" \
     --jd jd.txt --profile profile.txt \
     --bank question_bank.sample.json --out interview_set.json
   ```

### Command-line options

| Option | Required | Description |
|---|---|---|
| `--role` | Yes | Job title, for example `"Data Analyst Intern"` |
| `--jd` | Yes | Path to the job description text file |
| `--profile` | Yes | Path to the intern profile text file (resume, projects, coursework) |
| `--bank` | Yes | Path to the question bank JSON |
| `--out` | No | Output file, default `interview_set.json` |
| `--backend` | No | `anthropic` (default) or `llama` |
| `--dry-run` | No | Print the prompt and exit without calling a model |

## Input formats

### Question bank (JSON)

A list of tagged questions. The more you tag, the better the retrieval. Aim for 100 or more.

```json
{
  "question": "How would you find the top 5 customers by total order value?",
  "type": "technical",
  "skill": "sql",
  "difficulty": "easy",
  "rubric": "Joins tables, groups by customer, sums order value, orders descending, limits to 5."
}
```

| Field | Values |
|---|---|
| `type` | `technical`, `behavioral`, `project_based` |
| `skill` | Free text, ideally matching terms in `SKILL_VOCAB` |
| `difficulty` | `easy`, `medium`, `stretch` |
| `rubric` | What a good answer covers |

### Job description and profile

Plain text files. Remove names and personal details from profiles where you can. The tool also strips common PII patterns, but it is a safety net rather than a guarantee.

## Output format

```json
{
  "role": "Data Analyst Intern",
  "skills": ["python", "sql", "statistics"],
  "status": "ready_for_review",
  "validation_issues": [],
  "questions": {
    "technical": [
      {
        "question": "...",
        "skill": "sql",
        "difficulty": "easy",
        "rubric": "..."
      }
    ],
    "behavioral": [],
    "project_based": []
  }
}
```

Each set contains **5 technical** (2 easy, 2 medium, 1 stretch), **4 behavioral** (STAR-style), and **2 project-based** questions, each with a rubric.

If `status` is `needs_human_attention`, check `validation_issues` for what the validator could not resolve after retries.

## Configuration

Settings live at the top of `generator.py`.

| Setting | Default | Purpose |
|---|---|---|
| `LLM_MODEL` | `claude-sonnet-5-5` | Model name. Override with the `LLM_MODEL` environment variable |
| `EMBED_MODEL` | `all-MiniLM-L6-v2` | Embedding model for retrieval and duplicate checks |
| `DUP_THRESHOLD` | `0.90` | Cosine similarity above this counts as a duplicate |
| `MAX_RETRIES` | `2` | Regeneration attempts after a failed validation |
| `TOP_K_REFERENCES` | `8` | Number of bank questions passed to the model as references |
| `SKILL_VOCAB` | See file | Skills the extractor looks for in the JD |
| `BANNED_PATTERNS` | See file | Regexes for topics that must not appear in questions |

## Using a different model

The generator accepts any function that takes a prompt string and returns text.

**Local LLaMA** (requires `pip install transformers torch`):

```bash
export LOCAL_MODEL=meta-llama/Llama-3.1-8B-Instruct
python generator.py ... --backend llama
```

**Any other provider:** add a function to `generator.py` and pass it to `generate_interview_set`:

```python
def my_llm(prompt: str) -> str:
    ...  # call your API and return the response text
```

## Validation checks

| Check | What it catches |
|---|---|
| Schema and counts | Missing fields, wrong number of questions, invalid difficulty |
| Duplicate detection | Questions too similar to the bank or to each other |
| Skill coverage | JD skills with no matching question |
| Protected-topic filter | Questions about age, family status, religion, nationality, gender, disability, and similar |

When a check fails, the issues are sent back to the model and it tries again, up to `MAX_RETRIES` times.

## Limitations

- **The bias filter is keyword-based.** It catches obvious problems but not subtle ones. Always have a person review the set.
- **Skill extraction is vocabulary-based.** Skills missing from `SKILL_VOCAB` will not be detected, so extend it for your domain.
- **Skill coverage uses string matching.** It can miss a covered skill phrased differently.
- **Quality depends on your bank.** A small or untagged bank gives weaker references.
- **PII stripping is pattern-based.** It will not catch every identifying detail, such as names inside free text.

## Fairness and responsible use

- Use the same question structure and rubrics across candidates for the same role.
- Do not use generated questions to evaluate protected characteristics.
- Treat the output as a draft. Interviewers should approve or edit every set.
- Follow your local hiring and data-protection laws when storing candidate data.

## Roadmap ideas

- Web UI for interviewers to approve and edit questions
- Export to PDF or Word interview packs
- Log interviewer edits to build a fine-tuning dataset
- Difficulty classifier or second-pass LLM check
- Embedding-based skill coverage check
- Score tracking to see which questions best predict intern performance


