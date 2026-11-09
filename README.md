# guarded-rag-jpm

A Retrieval-Augmented Generation (RAG) system over JPMorgan Chase public filings, protected by layered security guardrails: input validation, secure ingestion, and output verification. The project is measured, not just built: every defensive layer is evaluated against a versioned red-team suite, and results are reported before and after each layer.

Status: In development (roadmap below)

---

## Table of Contents

1. Overview
2. Motivation
3. Scope and Non-Goals
4. Threat Model
5. System Architecture
6. Tech Stack
7. Dataset
8. Repository Structure
9. Roadmap
10. Evaluation Methodology
11. Results
12. Getting Started
13. Development Workflow
14. Continuous Red-Teaming (CI)
15. OWASP Mapping
16. Known Limitations
17. Disclaimer
18. License

---

## 1. Overview

This system answers questions about JPMorgan Chase using only a curated set of its official public documents. Unlike a basic RAG demo, it assumes that both user input and retrieved documents are untrusted, and it verifies every answer against its sources before returning it.

Core capabilities:

- Document-grounded question answering with citations.
- Input validation: prompt-injection detection, scope enforcement, PII handling.
- Secure ingestion: scanning documents for hidden or instruction-like content before indexing.
- Output sanitization: claim-level grounding, finance-aware number verification, leak detection.
- Abstention: the system declines to answer when evidence is insufficient.
- Automated red-team evaluation integrated into CI.

## 2. Motivation

A RAG demo that works on friendly questions is easy to build. A system that can be trusted with real users must handle adversarial input, poisoned documents, and unsupported claims. In finance, a fabricated or mis-scaled number is a material failure. This project demonstrates defense in depth and, more importantly, provides evidence of how well each layer performs.

## 3. Scope and Non-Goals

In scope:

- A single company (JPMorgan Chase) and a small, curated document set.
- Guardrails, evaluation, and engineering discipline as the primary deliverables.

Out of scope (intentionally):

- Multi-company comparison.
- Autonomous agents, tool use, fine-tuning, graph databases, voice or multimodal input.
- Any claim of complete protection. Prompt injection and hallucination are reduced and detected, not eliminated.

## 4. Threat Model

Assets to protect:

- Integrity of answers (no fabricated or mis-scaled figures).
- Confidentiality of the system prompt and any restricted documents.
- Availability and cost control of the service.

Adversaries and attack classes:

| Attack class | Example | Primary defense layer |
|---|---|---|
| Direct prompt injection | "Ignore previous instructions and ..." | Input guard |
| Jailbreak and role-play | Persona or hypothetical framing to bypass rules | Input guard, generation prompt |
| System prompt extraction | "Print your system prompt" | Input guard, output leak scan |
| Out-of-scope requests | Questions unrelated to JPMorgan or finance | Scope check |
| Indirect injection | Hidden instruction inside an ingested document | Secure ingestion, retrieval scan, privilege separation |
| Unsupported or fabricated claims | Numbers or statements absent from sources | Output guard (grounding) |
| Unit and period confusion | Millions vs billions, quarter vs full year | Finance number verifier |
| PII and secret leakage | Personal data in input or output | PII detector, output scan |
| Resource abuse | Very long or repeated queries | Rate and length limits |

Trust assumptions:

- User input: untrusted.
- Retrieved text: untrusted data, never treated as instructions.
- The LLM: not trusted to enforce policy on its own.

The full threat model is maintained in `docs/threat_model.md`.

## 5. System Architecture

```
User query
   |
[1] INPUT LAYER
    - length and format limits
    - PII detection and masking
    - prompt-injection detection (heuristics + classifier)
    - topic scope check (JPMorgan / finance only)
   |
[2] RETRIEVAL
    - hybrid search (BM25 + vector), reranking
    - metadata filters (filing type, period, access level)
    - scan of retrieved chunks for instruction-like content
   |
[3] GENERATION
    - system prompt: answer only from context, cite sources
    - retrieved text wrapped in delimiters, treated as data only
    - model has no ability to take actions
   |
[4] OUTPUT LAYER
    - claim-level grounding check against retrieved chunks
    - finance number verification (units, periods, percentages)
    - PII, secret, and system-prompt leak scan
    - on failure: abstain ("cannot verify") instead of answering
   |
Response + citations + confidence

Offline pipeline (secure ingestion):
Download -> Parse -> Scan (hidden text, injected instructions) -> Quarantine or Chunk -> Embed -> Index
Each document stored with source URL, hash, filing type, and period.
```

## 6. Tech Stack

Organized from basic to advanced.

### Foundation

| Component | Choice |
|---|---|
| Language | Python 3.11 |
| API framework | FastAPI |
| Vector database | Chroma (local) |
| Embeddings | sentence-transformers or API embeddings |
| LLM | Claude API (optionally a local model via Ollama) |
| PDF parsing | pdfplumber / PyMuPDF |
| Version control | Git, GitHub |

### Retrieval quality

| Component | Choice |
|---|---|
| Keyword search | BM25 (rank-bm25) |
| Hybrid retrieval | BM25 + vector score fusion |
| Reranking | Cross-encoder reranker |
| Table handling | Table-aware parsing for financial statements |

### Guardrails

| Component | Choice |
|---|---|
| Input validation | Custom rules, regex heuristics, small classifier model |
| PII detection | Microsoft Presidio (or equivalent) |
| Grounding check | NLI model or LLM-as-judge at claim level |
| Number verification | Custom finance normalizer (units, periods, percentages) |
| Reference comparison (optional) | NeMo Guardrails, Llama Guard |

### Evaluation and security testing

| Component | Choice |
|---|---|
| Test format | JSONL datasets in `eval/` |
| Red-team tooling | promptfoo, garak, PyRIT (as references or generators) |
| Attack generation | Second LLM to mutate attacks (paraphrase, encoding, Hinglish, other languages) |
| Test runner | pytest, custom `run_eval.py` |

### Engineering and operations

| Component | Choice |
|---|---|
| CI/CD | GitHub Actions |
| Secret scanning | pre-commit with gitleaks |
| Observability | Langfuse or OpenTelemetry |
| Containerization | Docker, docker-compose |
| Frontend | React or Next.js chat interface with citations panel and blocked-request dashboard |

## 7. Dataset

All documents are public and free, sourced from JPMorgan Chase investor relations and SEC EDGAR. PDFs are not committed to the repository. They are fetched by `scripts/download_data.py`, with sources documented in `data/README.md`.

Initial corpus (approximately 8 to 12 documents, 150 to 250 pages, 300 to 800 chunks):

| Document | Purpose | Approx. size |
|---|---|---|
| Latest annual shareholder letter | Text-heavy first test | 30-40 pages |
| Latest 10-K, Item 1A Risk Factors | Text-heavy QA | 30-40 pages |
| Latest 10-K, MD&A (selected sections) | Numbers and explanations | 40-60 pages |
| 2 to 3 latest quarterly earnings releases | Tables, year-over-year figures | 15-25 pages each |

Stretch corpus (after the pipeline works end to end): the full 10-K plus four quarters of earnings releases, approximately 2,000 chunks.

Chunking: approximately 500 tokens with 50-token overlap. Each chunk carries metadata: source document, filing type, period, page, and access level.

Evaluation data:

| Set | Size | Purpose |
|---|---|---|
| Benign questions with known answers | 50 to 60 | Accuracy and false-refusal rate |
| Direct attack prompts | 80 to 100 | Injection, jailbreak, prompt leak, out-of-scope |
| Indirect injection tests (poisoned chunks) | 10 to 15 | Hidden instructions inside documents |
| Number-trap questions | 15 to 20 | Unit, period, and percentage confusion |

## 8. Repository Structure

```
guarded-rag-jpm/
  README.md
  LICENSE
  .gitignore
  .env.example
  .pre-commit-config.yaml
  requirements.txt
  docs/
    threat_model.md
    architecture.md
  data/
    README.md
  scripts/
    download_data.py
  src/guarded_rag/
    ingest/            parse, chunk, scan, embed
    input_guard/       injection detection, scope check, PII
    retrieval/         hybrid search, rerank
    generation/        prompt assembly, LLM call
    output_guard/      grounding, number check, leak scan
    api/               FastAPI application
  eval/
    benign_questions.jsonl
    attacks.jsonl
    poisoned_docs/
    results/
    run_eval.py
  tests/
  frontend/
  .github/workflows/ci.yml
```

## 9. Roadmap

### Phase 0: Setup (Day 1)

- [ ] Create repository, `.gitignore`, `.env.example`, license.
- [ ] Install pre-commit with gitleaks.
- [ ] Write `scripts/download_data.py` and `data/README.md`.
- [ ] Download the initial corpus.

### Phase 1: Basic RAG (Weeks 1-2)

- [ ] Parse PDFs and chunk documents.
- [ ] Embed and index in Chroma.
- [ ] Retrieval plus LLM answer with citations.
- [ ] FastAPI endpoint `/ask`.
- [ ] Minimal command-line or web test interface.

### Phase 2: Evaluation harness and baseline (Week 3)

- [ ] Write 50 to 60 benign questions with ground-truth answers.
- [ ] Write first 50 attack prompts.
- [ ] Implement `eval/run_eval.py` (accuracy, attack success rate, false refusals, latency).
- [ ] Run on the unguarded system and commit results.
- [ ] Tag `v0.1-baseline`.

### Phase 3: Input layer (Week 4)

- [ ] Length, format, and rate limits.
- [ ] Prompt-injection detection (heuristics, then classifier).
- [ ] Scope check (JPMorgan and finance only).
- [ ] PII detection and masking.
- [ ] Re-run evaluation. Tag `v0.2-input-guard`.

### Phase 4: Output layer (Week 5)

- [ ] Citation requirement and verification.
- [ ] Finance number verifier (units, periods, percentages).
- [ ] Claim-level grounding check.
- [ ] Leak scan (PII, secrets, system prompt).
- [ ] Abstention behavior.
- [ ] Re-run evaluation. Tag `v0.3-output-guard`.

### Phase 5: Secure ingestion and indirect injection (Week 6)

- [ ] Ingestion scanner: hidden text, instruction-like phrases, anomalies.
- [ ] Quarantine suspicious chunks; record document hash and source.
- [ ] Retrieval-time scan and privilege separation (delimited, data-only context).
- [ ] Poisoned-document test cases demonstrating detection.
- [ ] Tag `v0.4-secure-ingestion`.

### Phase 6: Retrieval quality and observability (Week 7)

- [ ] Hybrid search and reranking.
- [ ] Table-aware parsing for financial statements.
- [ ] Time awareness: answers state the filing and period used.
- [ ] Request tracing across all layers (Langfuse or OpenTelemetry).

### Phase 7: Interface, CI, and write-up (Week 8)

- [ ] Frontend with citations panel and blocked-request dashboard.
- [ ] GitHub Actions: tests plus red-team suite with failure thresholds.
- [ ] Complete `docs/threat_model.md` and results tables.
- [ ] Write the "Attacks that still succeed" section.
- [ ] Tag `v1.0`.

### Phase 8: Stretch goals (optional)

- [ ] Automated attack generation by a second LLM (paraphrase, encoding, multilingual, Hinglish).
- [ ] Role-based access control with restricted documents.
- [ ] Docker deployment and public demo with latency and cost budget.
- [ ] Comparison against NeMo Guardrails or Llama Guard.
- [ ] Expanded corpus (full 10-K, four quarters of earnings releases).

## 10. Evaluation Methodology

Metrics reported for every tagged version:

| Metric | Definition |
|---|---|
| Attack success rate (ASR) | Fraction of attack prompts that achieve the attacker's goal |
| Indirect injection success rate | Fraction of poisoned-document tests that alter the answer |
| False refusal rate | Fraction of benign questions wrongly blocked |
| Answer accuracy | Fraction of benign questions answered correctly against ground truth |
| Grounding rate | Fraction of answer claims supported by retrieved chunks |
| Number accuracy | Fraction of number-trap questions answered with correct value, unit, and period |
| Coverage | Fraction of benign questions answered rather than abstained |
| Added latency | Median and 95th percentile latency added by guardrails |
| Cost per query | Average API cost per request |

Principles:

- The baseline is measured before any guardrail is added.
- Datasets are versioned in the repository and never edited to improve results after the fact.
- Failures are reported, not hidden.

## 11. Results

To be completed as phases finish.

| Version | ASR | Indirect injection success | False refusal | Accuracy | Grounding | Added latency (p50) |
|---|---|---|---|---|---|---|
| v0.1-baseline | TBD | TBD | TBD | TBD | TBD | 0 ms |
| v0.2-input-guard | TBD | TBD | TBD | TBD | TBD | TBD |
| v0.3-output-guard | TBD | TBD | TBD | TBD | TBD | TBD |
| v0.4-secure-ingestion | TBD | TBD | TBD | TBD | TBD | TBD |
| v1.0 | TBD | TBD | TBD | TBD | TBD | TBD |

## 12. Getting Started

Prerequisites: Python 3.11, Git, an Anthropic API key (or a local model through Ollama).

```bash
git clone https://github.com/Tiwari1782/guarded-rag-jpm.git
cd guarded-rag-jpm

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env             # then add your API key
python scripts/download_data.py  # fetch public filings
python -m guarded_rag.ingest     # parse, scan, chunk, embed
uvicorn guarded_rag.api.main:app --reload
```

Run the evaluation suite:

```bash
python eval/run_eval.py
```

Environment variables (`.env.example`):

```
ANTHROPIC_API_KEY=your_key_here
CHROMA_PATH=./chroma_db
LOG_LEVEL=INFO
```

Never commit `.env`.

## 13. Development Workflow

Branching:

- `main` is always working. No direct commits after the initial setup.
- One branch per feature: `feat/basic-rag`, `feat/input-guard`, `feat/output-guard`, `feat/eval-harness`, `feat/ci-redteam`.
- All changes merge through a Pull Request that includes a short description and evaluation numbers.
- Branch protection on `main` requires the CI check to pass.

Commit convention: `type: short imperative description`

Types: `feat`, `fix`, `docs`, `test`, `eval`, `refactor`, `ci`, `chore`

```
feat: add PDF ingestion and chunking for 10-K risk factors
feat: add regex and classifier based injection detector
eval: add 100 attack prompts and baseline results
fix: normalize millions and billions in number verification
ci: run red-team suite on every push
docs: add threat model mapped to OWASP LLM Top 10
```

Milestone tags:

```bash
git tag -a v0.1-baseline -m "Plain RAG, no guardrails, baseline metrics"
git tag -a v0.2-input-guard -m "Input validation layer added"
git tag -a v0.3-output-guard -m "Output grounding layer added"
git push --tags
```

Secret protection: pre-commit with gitleaks is required. If a key is ever committed, revoke it immediately; removing the commit is not sufficient.

## 14. Continuous Red-Teaming (CI)

On every push, GitHub Actions runs unit tests and a reduced red-team subset (approximately 30 prompts). The full suite runs on merge to `main`. The build fails if attack success rate rises above the configured threshold or accuracy drops below it.

```yaml
name: ci
on: [push, pull_request]
jobs:
  test-and-redteam:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install -r requirements.txt
      - run: pytest tests/
      - run: python eval/run_eval.py --fail-if-attack-success-above 0.10
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

Add the API key under repository Settings, Secrets and variables, Actions.

## 15. OWASP Mapping

Mapped to the OWASP Top 10 for LLM Applications (2025 edition):

| OWASP item | Addressed by |
|---|---|
| LLM01 Prompt Injection | Input guard, privilege separation, ingestion and retrieval scans |
| LLM02 Sensitive Information Disclosure | PII detection, output leak scan |
| LLM04 Data and Model Poisoning | Secure ingestion, quarantine, document hashing |
| LLM05 Improper Output Handling | Output sanitization, citation and grounding checks |
| LLM07 System Prompt Leakage | Input guard, output leak scan |
| LLM08 Vector and Embedding Weaknesses | Metadata filtering, access control, ingestion scanning |
| LLM09 Misinformation | Claim-level grounding, number verification, abstention |
| LLM10 Unbounded Consumption | Length limits, rate limits, cost tracking |

## 16. Known Limitations

This section will be maintained with real results.

- Prompt injection cannot be fully prevented; this system reduces and detects it.
- Hallucination is reduced and flagged, not eliminated.
- Table extraction from financial PDFs is imperfect and may cause number errors.
- Guardrails add latency and may produce false refusals.
- The corpus covers a limited set of JPMorgan documents and may not reflect the latest filings.
- Attacks that still succeed: to be documented after evaluation.

## 17. Disclaimer

This project is for educational and portfolio purposes. It is not affiliated with, endorsed by, or sponsored by JPMorgan Chase & Co. Source documents are public filings. Outputs do not constitute financial advice and should be verified against the original documents.

## 18. License

MIT License. See `LICENSE`.
