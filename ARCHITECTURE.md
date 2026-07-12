# Architecture — Library Search Using Agentic AI Model

## 1. Overview

This project is a **local, agentic AI code-auditing pipeline**. Its purpose is to
answer a single enterprise question at scale:

> "Where, and *why*, is a given library (e.g. `numpy`) used across our entire
> repository estate?"

Instead of relying on a hosted LLM API, the system runs a **local code-specialised
language model** (`Qwen/Qwen2.5-Coder-1.5B-Instruct`) on the analyst's own GPU. It
clones a fleet of repositories, statically locates every file that references the
target library, and then asks the local model to explain the *specific* purpose of
each usage in its surrounding code context. The results are streamed to a plain-text
audit log and compiled into a Word document (later exported to PDF).

The system is "agentic" in that the model is given a constrained role
(a *precise code auditor*), strict output rules, and is invoked autonomously in a
loop over hundreds of files across many repositories and branches — with no
human in the loop per file.

---

## 2. Pipeline Stages

The repository is organised as a set of standalone scripts, each owning one phase
of the pipeline. They are meant to be run in sequence.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          ENVIRONMENT PREPARATION                               │
│                                                                                │
│   dummy repos.py            upload to my repo.py                               │
│   (build a test estate)     (mirror repos into your own account)               │
│         │                            │                                         │
│         ▼                            ▼                                         │
│   Forks ~100 public          Re-homes locally downloaded repos under           │
│   Python repos into          your GitHub account so you have a stable,         │
│   amirthap-test-env          self-owned test corpus to audit                   │
└──────────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                       PHASE 1 — ACQUISITION  (clone.py)                        │
│                                                                                │
│   Hard-coded list of repo URLs  ──git clone──▶  C:\Audit\Repos\<repo>          │
│   (idempotent: skips repos already present on disk)                            │
└──────────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                     PHASE 2 — AUDIT  (modelx.py)  ★ core                        │
│                                                                                │
│   Load local model ▶ prompt for target library ▶ walk repos/branches/files    │
│        ▶ static match ▶ dedupe ▶ LLM analysis ▶ write log + .docx              │
└──────────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                              OUTPUTS (artifacts)                               │
│                                                                                │
│   audit_log_<lib>.txt   —  chronological scan trace                            │
│   Final_Audit_<lib>.docx —  structured per-file findings report                │
│   Final_Audit_<lib>.pdf  —  distributable export of the report                 │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Components

### 3.1 `dummy repos.py` — Test-estate builder *(optional, setup-only)*
- Authenticates to the GitHub REST API with a Personal Access Token
  (read via `getpass`, never hard-coded).
- Searches for real public Python repositories filtered by size and star count
  (`size:100..5000 stars:10..500`).
- **Forks** the matching repositories into the working account, pausing 3s between
  calls to respect GitHub rate limits / abuse detection.
- Produces a realistic corpus of ~100 repos to exercise the auditor against.

### 3.2 `upload to my repo.py` — Repo re-homing helper *(optional, setup-only)*
- Iterates over locally downloaded repository folders (`downloaded_repos/`).
- For each folder: creates a matching empty repo on the user's account
  (`201 Created`, tolerating `422` "already exists"), rewrites the `origin`
  remote, and `git push --all` to publish it.
- Enforces a **15-second cooldown** between pushes to avoid being flagged as spam.
- Net effect: gives the analyst a self-owned, stable copy of the corpus that will
  not disappear if the upstream source changes.

### 3.3 `clone.py` — Acquisition (Phase 1)
- Holds an explicit list of repository URLs (the audit scope).
- Clones each into `C:\Audit\Repos\<repo-name>` using a **standard (non-mirror)**
  clone.
- **Idempotent**: skips any repository already present on disk, so the phase can be
  re-run safely after interruption.

### 3.4 `modelx.py` — Audit engine (Phase 2) ★
The heart of the system. Broken into four logical blocks:

1. **Configuration & model setup**
   - Sets `HF_HUB_DISABLE_SYMLINKS_WARNING` and
     `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` to keep GPU memory
     fragmentation under control.
   - Loads tokenizer + causal LM (`Qwen2.5-Coder-1.5B-Instruct`) with
     `device_map="auto"` and `torch_dtype="auto"` into a local `model_storage/`
     cache directory.
   - Prompts the analyst for the library to audit (`AUDIT_LIB`), and derives
     absolute paths for the log (`audit_log_<lib>.txt`) and report
     (`Final_Audit_<lib>.docx`).

2. **Local AI analysis function — `analyze_locally()`**
   - Truncates any file over ~15,000 chars (~4,000 tokens) to prevent GPU
     **VRAM out-of-memory** errors.
   - Builds a tightly-constrained prompt: the model must return only the
     *enclosing block* and the *specific purpose* of the usage, and is explicitly
     forbidden from giving generic definitions of the library.
   - Runs generation under `torch.no_grad()` with `max_new_tokens=256`, decodes
     only the newly generated tokens, and frees input/output tensors immediately.

3. **Main execution loop**
   - Walks every repo directory under `C:\Audit\Repos`.
   - For each repo: `git fetch --all`, enumerates **all remote branches**
     (stripping `origin/` and skipping `HEAD`), and checks each branch out with
     `git checkout -f`.
   - Walks every `.py` file, reads it defensively (`errors='ignore'`), and:
     - **De-duplicates** by MD5 content hash so identical files across branches are
       analysed only once.
     - Matches files that textually contain `AUDIT_LIB`, records the matching line
       numbers (capped at 10), logs a `[MATCH]`, and calls `analyze_locally()`.
     - Appends a heading + line list + model analysis to the Word document and
       **saves after every file** (crash-resilient incremental persistence).
     - Flushes CUDA cache after each analysis (and on error) to keep VRAM clean.

4. **Resilience & teardown**
   - `KeyboardInterrupt` is caught so a manual stop still saves progress.
   - `finally` guarantees a final `doc.save()` and a closing log line.

### 3.5 `instructions.txt` — Operator runbook
- One-time PowerShell setup: temporarily bypass execution policy for the session
  and activate the virtual environment.
- Handy `gh` command to dump every repo URL in the test org for populating
  `clone.py`.

---

## 4. Data Flow

```
GitHub (public repos)
      │  fork / clone
      ▼
C:\Audit\Repos\<repo>\**\*.py   ──read──▶  MD5 dedupe  ──match "AUDIT_LIB"──▶
      │
      ▼
constrained prompt  ──▶  Qwen2.5-Coder-1.5B  (local GPU)  ──▶  analysis text
      │                                                          │
      └──────────────▶  audit_log_<lib>.txt  (trace)             │
                                                                  ▼
                                        Final_Audit_<lib>.docx  ──export──▶  .pdf
```

---

## 5. Key Design Decisions

| Decision | Rationale |
|---|---|
| **Local model, not a hosted API** | Source code never leaves the machine — critical for auditing private/enterprise code. No per-token cost, no rate limits. |
| **Small code-specialised model (1.5B)** | Fits on a modest GPU; the task (localised usage explanation) is narrow enough that a small coder model suffices. |
| **Static match *then* LLM** | The cheap `AUDIT_LIB in content` filter avoids invoking the model on irrelevant files, drastically cutting GPU work. |
| **MD5 content de-duplication** | Multi-branch repos contain many identical files; hashing prevents re-analysing the same content. |
| **Save-after-every-file** | Audits over hundreds of repos are long-running; incremental saves make the run crash- and interrupt-resilient. |
| **Aggressive VRAM hygiene** | Truncation, `no_grad`, tensor deletion, and `empty_cache()` keep a small GPU from OOMing across a long loop. |
| **Constrained "auditor" prompt** | Forbidding generic definitions forces context-specific, actionable findings rather than boilerplate. |

---

## 6. Technology Stack

- **Language:** Python 3
- **AI / ML:** `transformers`, `torch` (CUDA), model
  `Qwen/Qwen2.5-Coder-1.5B-Instruct`
- **VCS automation:** `git` CLI via `subprocess`, GitHub REST API via `requests`
- **Reporting:** `python-docx` (`Document`) → `.docx`, exported to `.pdf`
- **Std lib:** `os`, `subprocess`, `hashlib`, `datetime`, `getpass`, `time`
- **Platform:** Windows (paths such as `C:\Audit\Repos`, PowerShell venv activation)

---

## 7. Operational Notes & Limitations

- **Matching is textual, not semantic.** `AUDIT_LIB in line` will match comments,
  strings, and substrings (e.g. auditing `re` matches `pandas.read_csv`). Findings
  should be treated as *candidates* for the model to interpret, not ground truth.
- **Branch checkout is destructive** (`git checkout -f`) — intended for throwaway
  audit clones, not working copies.
- **Hard-coded paths** (`C:\Audit\Repos`) and a hard-coded repo list couple the
  scripts to one environment; these are the natural first parameters to externalise.
- **Secrets** are read interactively via `getpass` and never persisted — tokens
  should continue to be handled this way.
- **Scale** is bounded by disk (full clones of ~100 repos) and GPU VRAM
  (mitigated by truncation + cache flushing).
