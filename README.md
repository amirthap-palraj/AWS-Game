# Library Search Using Agentic AI Model

An offline, **agentic AI code-auditing tool** that scans a fleet of Git repositories
to find *where* and *why* a given library (for example `numpy`) is used — and
explains each usage in plain English using a **local** code language model.

No source code ever leaves your machine: the analysis runs on a locally hosted
`Qwen/Qwen2.5-Coder-1.5B-Instruct` model on your own GPU, and the results are
compiled into a Word / PDF audit report.

> Example run included in this repo: an audit of **`numpy`** across ~100 repositories
> (`audit_log_numpy.txt`, `Final_Audit_numpy.docx`, `Final_Audit_numpy.pdf`).

---

## ✨ What it does

1. **Clones** a defined list of repositories to a local audit workspace.
2. **Walks every branch** of every repo and scans all `.py` files.
3. **Statically matches** files that reference your target library and records the
   exact line numbers.
4. **Asks a local LLM** to identify the *enclosing function/class* and the
   *specific purpose* of each usage — not a generic definition.
5. **Reports** everything to a live text log and an incrementally-saved Word
   document.

---

## 📂 Repository contents

| File | Role |
|---|---|
| `clone.py` | **Phase 1 — Acquisition.** Clones the target repositories into `C:\Audit\Repos` (idempotent; skips repos already on disk). |
| `modelx.py` | **Phase 2 — Audit engine ★.** Loads the local model, prompts for the library to audit, scans repos/branches/files, and writes the log + report. |
| `dummy repos.py` | *Setup helper.* Forks ~100 real public Python repos into a test org to build an audit corpus. |
| `upload to my repo.py` | *Setup helper.* Re-homes locally downloaded repos under your own GitHub account. |
| `instructions.txt` | Operator runbook (PowerShell venv setup, `gh` repo-listing command). |
| `audit_log_numpy.txt` | Sample output — chronological scan trace. |
| `Final_Audit_numpy.docx` / `.pdf` | Sample output — structured findings report. |

See [`ARCHITECTURE.md`](ARCHITECTURE.md) for a full breakdown of the pipeline,
data flow, and design decisions.

---

## 🚀 Quick start

### Prerequisites
- **Windows** with Python 3.9+
- An NVIDIA **GPU with CUDA** (the model auto-selects the device; CPU will be slow)
- **Git** and (for the setup helpers) the **GitHub CLI** / a Personal Access Token
- Enough disk for full clones of every repo in scope

### 1. Set up the environment
```powershell
# Allow scripts for this PowerShell session only
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass

# Create & activate a virtual environment
python -m venv venv
.\venv\Scripts\Activate.ps1

# Install dependencies
pip install torch transformers python-docx requests
```
> Install the CUDA-enabled build of `torch` that matches your GPU / driver — see
> the [PyTorch install guide](https://pytorch.org/get-started/locally/).

### 2. (Optional) Build a test corpus
```powershell
python "dummy repos.py"          # fork ~100 public Python repos
python "upload to my repo.py"    # re-home downloaded repos to your account
```
Both prompt for a GitHub token securely (input hidden) — nothing is stored.

### 3. Phase 1 — Clone the repositories
Edit the `repo_urls` list in [`clone.py`](clone.py) to your audit scope, then:
```powershell
python clone.py
```
Repositories are cloned into `C:\Audit\Repos`.

> Tip: dump every repo URL from an org with
> `gh repo list <org> --limit 200 --json url --jq '.[].url'`.

### 4. Phase 2 — Run the audit
```powershell
python modelx.py
```
You'll be prompted:
```
Which library are we auditing?  numpy
```
The model loads, the scan runs, and progress streams to the console and to
`audit_log_<library>.txt`.

### 5. Collect the results
- `audit_log_<library>.txt` — full scan trace (repos, branches, matches).
- `Final_Audit_<library>.docx` — per-file findings (enclosing block + purpose),
  saved incrementally so partial runs are never lost.

---

## ⚙️ Configuration

Key settings live at the top of `modelx.py`:

| Setting | Default | Description |
|---|---|---|
| `TARGET_DIR` | `C:\Audit\Repos` | Where cloned repos are read from. |
| `MODEL_ID` | `Qwen/Qwen2.5-Coder-1.5B-Instruct` | The local code model. |
| `LOCAL_MODEL_PATH` | `./model_storage` | Local Hugging Face cache directory. |
| *(truncation limit)* | `15000` chars | Files larger than this are truncated to protect GPU VRAM. |
| *(generation)* | `max_new_tokens=256` | Length of each analysis. |

---

## 🧠 How the analysis is kept accurate & efficient

- **Static match before LLM** — the cheap `library in content` filter avoids
  invoking the model on irrelevant files.
- **MD5 de-duplication** — identical files across branches are analysed only once.
- **Constrained prompt** — the model is told to act as a *precise code auditor* and
  is explicitly forbidden from giving generic library definitions.
- **VRAM hygiene** — truncation, `torch.no_grad()`, tensor deletion, and
  `torch.cuda.empty_cache()` keep long runs from running out of GPU memory.
- **Crash resilience** — the report is saved after every file, and `KeyboardInterrupt`
  still saves progress on exit.

---

## ⚠️ Notes & limitations

- Matching is **textual, not semantic** — it can match comments, strings, or
  substrings. Treat findings as candidates the model then interprets.
- `git checkout -f` is **destructive** — run only against throwaway audit clones,
  never a working copy.
- Paths and the repo list are **hard-coded for a Windows environment** — adjust
  `TARGET_DIR` and the `repo_urls`/`base_dir` values for your setup.
- GitHub tokens are handled **interactively via `getpass`** and never written to
  disk — keep it that way.

---

## 🛠️ Tech stack

Python · `transformers` · `torch` (CUDA) · `Qwen2.5-Coder-1.5B-Instruct` ·
`python-docx` · GitHub REST API (`requests`) · Git CLI

---

## 📄 License

No license file is currently included. Add one (e.g. MIT) before distributing.
