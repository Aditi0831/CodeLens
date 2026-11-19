# CodeLens: College Project Documentation

## Project Title
**CodeLens: Intelligent Code Review and Quality Audit System Using Hybrid Static Analysis and Local LLMs**

## Abstract
CodeLens is an automated code review and software quality audit system designed to bridge the gap between deterministic static analysis and semantic reasoning using local large language models. The system analyzes Python source code and multi-file archives, generates immutable content-hashed snapshots, and produces evidence-grounded findings categorized by severity and vulnerability type. It integrates real static analyzers (Ruff and Bandit) alongside an in-depth AST metrics engine calculating McCabe Cyclomatic Complexity, Cognitive Complexity, Halstead Metrics, and Maintainability Index. When configured with local coding models (Qwen2.5-Coder, DeepSeek-Coder, or Code Llama via Ollama), it offers automated patch generation with syntax verification, docstring generation, algorithm explanations, and test generation. All submitted code remains sandboxed and is never executed.

---

## 1. Problem Statement and Motivation
Peer code review is one of the most critical steps in the software engineering lifecycle for catching bugs and security vulnerabilities early. However, manual reviews are time-consuming and often miss subtle boundary bugs, resource leaks, or anti-patterns. Commercial AI assistants often require uploading proprietary source code to external third-party cloud servers, posing privacy and data-leak risks.

CodeLens solves this by:
1. Running completely locally: using local tools and local Ollama models with zero external network dependencies.
2. Grounding AI claims in verifiable code: every finding must cite exact file paths and source lines.
3. Combining static guarantees with AI reasoning: deterministic linting and AST metrics never fail even if an LLM is offline or times out.
4. Ensuring strict safety: uploaded code is parsed as data and is never executed.

---

## 2. System Architecture
CodeLens is structured as a modular monolith with an asynchronous job execution model:

```
[ User Browser / React 18 + Monaco ]
                 |  (HTTP REST / JSON)
                 v
       [ FastAPI Backend ] <-----> [ Database (PostgreSQL / SQLite) ]
                 |
                 +--> [ Job Queue (Redis RQ or Local ThreadPool) ]
                            |
                            v
                    [ Analysis Worker ]
                      |-- AST Parsing & Indexing
                      |-- AST Metrics (Cyclomatic, Halstead, MI)
                      |-- Ruff Linter Subprocess
                      |-- Bandit Security Scanner Subprocess
                      +-- Ollama Local Inference (Qwen / DeepSeek / CodeLlama)
```

### Key Modules:
- `backend/app/main.py`: REST API endpoints, upload handlers, job status polling, and multi-format report exports (JSON, Markdown, HTML).
- `backend/app/metrics.py`: Custom AST analysis engine for Cyclomatic Complexity, Cognitive Complexity, Halstead metrics, and code smells.
- `backend/app/analysis.py`: Subprocess runner for Ruff and Bandit, rule catalog mapping, and finding deduplication.
- `backend/app/provider.py`: Clean model provider interface with Ollama integration and offline mock simulation mode for viva demos.
- `backend/app/context.py`: Symbol-aware AST chunker that selects complete functions and imports without exceeding token budgets.
- `frontend/src/`: React 18 application with Monaco Editor, interactive line markers, diff views, and model comparison lab.

---

## 3. Project Demonstration Script (7-10 minutes)

### Step 1: System Overview (1 min)
- Explain the motivation: local-first, privacy-preserving code review with verifiable line evidence.
- Point out the tech stack: FastAPI, React + TypeScript + Monaco Editor, SQLite/PostgreSQL, Ruff, Bandit, and Ollama.

### Step 2: Running a Code Review (2 mins)
- Open the web interface at `http://127.0.0.1:5173`.
- Click **Load example** (loads `samples/review_me.py`).
- Select **Static** mode and click **Run review**.
- Walk through the findings:
  - Mutable default argument bug (Ruff rule B006) on line 3.
  - Subprocess call with `shell=True` security warning (Bandit test B602) on line 14.
- Click a finding to show how Monaco editor highlights the exact offending line and displays fix guidance.

### Step 3: Code Metrics and Health Score (2 mins)
- Demonstrate the Code Health Score: calculated from severity deductions, maintainability index, and complexity.
- Explain the AST metrics: Cyclomatic Complexity ($M = 1 + \text{decision points}$), Cognitive Complexity, and Halstead volume.

### Step 4: Multi-File Project Archive (1.5 mins)
- Click **Upload .py / .zip** and upload `samples/sample_project.zip`.
- Show how CodeLens safely extracts the archive, rejects malicious traversal paths, parses package modules, and allows file-by-file navigation.

### Step 5: AI Artifacts and Model Comparison (2 mins)
- Select the **Explain**, **Docs**, **Tests**, or **Changes** tabs.
- Show the generated diff view: comparing the original code with proposed refactored code side-by-side.
- Switch to the **Model Lab** tab and demonstrate sequential comparison between Qwen, DeepSeek, and Code Llama.

### Step 6: Export and History (1 min)
- Export the audit report in HTML and Markdown formats.
- Show the **Review history** tab to demonstrate that all reviews survive server restarts through database persistence.

---

## 4. Key Viva Questions and Answers

**Q1: What is the main innovation of CodeLens compared to simply using ChatGPT or GitHub Copilot?**
**Answer:** CodeLens is a local-first, hybrid system. Unlike generic chat models, CodeLens:
1. Runs entirely on the user's machine, keeping source code private.
2. Validates model claims: every finding is checked against source line numbers and quoted code to prevent hallucinations.
3. Integrates deterministic tools (Ruff, Bandit, and AST metrics) so users get guaranteed static security findings even if the model is unavailable.
4. Generates side-by-side visual diffs and formal software engineering metrics (Cyclomatic Complexity and Maintainability Index).

**Q2: How does CodeLens prevent executing untrusted submitted code?**
**Answer:** Submitted source code is stored strictly as text in the database and parsed into an Abstract Syntax Tree (AST) using Python's standard `ast` library. The application never imports, evaluates (`eval`), or executes user-submitted code in any runtime environment.

**Q3: How does the AST context selection work?**
**Answer:** Instead of arbitrary line-based chunking which breaks functions in half, CodeLens inspects the AST to extract complete semantic units (classes, functions, and imports). Top-level imports and target functions are prioritized to fit within the model context budget, and any omitted lines are transparently reported in coverage metrics.

**Q4: How do you handle model hallucinations in line numbers or findings?**
**Answer:** Every finding returned by an LLM passes through `validate_finding()`. The system verifies that the reported file exists, the line range is within file bounds, and the quoted evidence string actually exists in the source text. If a model hallucinates a non-existent line, it is flagged as `invalid_location` or `unsupported_evidence`.

**Q5: Why did you choose a hybrid approach instead of relying purely on an LLM?**
**Answer:** LLMs are probabilistic: they are great at explaining context and suggesting creative optimizations, but they can miss standard syntax errors or invent false issues. Static tools (Ruff and Bandit) are deterministic and reproducible. By combining them, CodeLens gives the developer verified linting/security findings plus the high-level explanations and test suggestions of an LLM.

**Q6: What software metrics are calculated and how?**
**Answer:** CodeLens calculates:
- McCabe Cyclomatic Complexity: $M = 1 + \text{decision points}$ (`if`, `for`, `while`, `except`, `bool_ops`).
- Cognitive Complexity: measures human reading difficulty by penalizing nested control structures.
- Halstead Metrics: calculates vocabulary, volume, difficulty, and estimated bugs based on distinct operators and operands.
- Maintainability Index: standard SEI formula combining Halstead Volume, Cyclomatic Complexity, and Source Lines of Code.

---

## 5. Division of Work
- **Backend & Static Analysis:** FastAPI application, database schema migrations, AST metrics engine, Ruff and Bandit integration.
- **Frontend & UI/UX:** React workspace, Monaco editor integration, diff viewer, filtering and history interfaces.
- **Model Integration & Prompts:** Ollama adapter, JSON schema structured output validation, offline simulation mode, Model Lab.
- **Testing & Packaging:** Pytest suite, Docker Compose setup, Windows automated startup script (`start.ps1`), documentation.
