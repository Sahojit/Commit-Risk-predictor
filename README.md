# Commit Risk Predictor

> **ML system that flags high-risk code commits before they reach production.**

Git commit-level risk scoring using code change embeddings to predict bug-introducing commits pre-merge. Catches bad deployments at the source — before they ever hit CI/CD.

---

## What It Does

Most bugs don't appear out of nowhere — they're introduced at a specific commit. This system analyzes every commit diff, converts it into semantic embeddings, and scores it for risk **before** the merge happens.

If a commit looks like past bug-introducing changes, it gets flagged. Simple.

---

## Architecture

```
Git Commit Diff
      │
      ▼
 Diff Extractor          ← Parses staged/committed changes
      │
      ▼
 Code Embedder           ← CodeBERT / sentence-transformers on diff hunks
      │
      ▼
 Risk Scoring Engine     ← Classifier trained on historical bug-labeled commits
      │
      ▼
 Risk Label + Score      ← LOW / MEDIUM / HIGH with confidence score
      │
      ▼
 Pre-merge Hook / CI     ← Block, warn, or annotate PR automatically
```

---

## Key Features

- **Semantic diff analysis** — understands *what* changed, not just *how much*
- **Code change embeddings** — uses CodeBERT to represent diff hunks as vectors
- **Risk classification** — trained on real commit history with bug labels
- **Pre-merge enforcement** — integrates as a Git pre-push hook or CI step
- **Explainable output** — surfaces the high-risk lines/hunks in the diff
- **Lightweight inference** — fast enough to run on every push

---

## Tech Stack

| Layer | Technology |
|---|---|
| Embedding Model | CodeBERT / `microsoft/codebert-base` |
| ML Framework | scikit-learn / PyTorch |
| Feature Extraction | `gitpython`, `unidiff` |
| Risk Classifier | Gradient Boosting / fine-tuned transformer head |
| Integration | Git hooks, GitHub Actions |
| Experiment Tracking | MLflow |

---

## How It Works

### 1. Feature Extraction
Each commit diff is broken into **hunks** — individual blocks of added/removed lines. Each hunk is tokenized and passed through CodeBERT to produce a 768-dim embedding.

### 2. Commit-Level Representation
Hunk embeddings are **pooled** (mean/max) into a single commit vector. Additional structural features are appended:
- Lines added / deleted
- Files changed
- Commit message length
- Time since last commit to same file
- Author commit history risk score

### 3. Risk Classification
The combined feature vector is fed into a classifier trained on labeled commits (bug-fix commits + their parent as negative samples from open-source repos).

Output:
```json
{
  "commit": "a3f92c1",
  "risk_score": 0.87,
  "risk_label": "HIGH",
  "flagged_files": ["src/auth/token_handler.py", "src/db/session.py"],
  "reason": "High semantic similarity to past authentication bug commits"
}
```

### 4. Pre-merge Gate
Integrates as:
- **Git pre-push hook** — runs locally before push
- **GitHub Actions step** — runs on PR open/update
- **CLI tool** — `commit-risk check HEAD`

---

## Quick Start

```bash
# Clone the repo
git clone https://github.com/Sahojit/commit-risk-predictor.git
cd commit-risk-predictor

# Install dependencies
pip install -r requirements.txt

# Install as a pre-push hook in your project
python setup_hook.py --repo /path/to/your/project

# Or run manually on any commit
python predict.py --commit a3f92c1
```

---

## Training Your Own Model

```bash
# 1. Collect commit data from a repo
python collect_commits.py --repo /path/to/repo --output data/commits.jsonl

# 2. Label commits (bug-introducing = 1, clean = 0)
python label_commits.py --input data/commits.jsonl

# 3. Extract embeddings
python embed_diffs.py --input data/commits.jsonl --output data/embeddings.npy

# 4. Train the classifier
python train.py --embeddings data/embeddings.npy --labels data/labels.npy

# 5. Evaluate
python evaluate.py --model models/risk_classifier.pkl
```

---

## Sample Output

```
$ commit-risk check HEAD

Analyzing commit: a3f92c1 — "refactor token refresh logic"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Risk Score  :  0.87  ████████░░  HIGH
  Files       :  3 changed  (+142 / -67)
  Flagged     :  src/auth/token_handler.py
                 src/db/session.py

  ⚠ This commit closely resembles patterns from 14 historical bug commits.
  Recommend: peer review before merge.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Dataset

Trained on labeled commits from open-source Python repositories. Bug-introducing commits identified via **SZZ algorithm** — linking bug-fix commits back to the commits that introduced the defect.

- ~50K total commits
- ~12% labeled as bug-introducing
- Balanced via stratified sampling

---

## Results

| Metric | Score |
|---|---|
| Precision (HIGH risk) | 0.81 |
| Recall (HIGH risk) | 0.74 |
| F1 Score | 0.77 |
| AUC-ROC | 0.89 |

---

## Use Cases

- **Pre-merge review prioritization** — focus reviewer attention on risky diffs
- **CI/CD gating** — block HIGH risk commits from auto-deploy pipelines
- **Developer feedback** — real-time risk score while coding
- **Audit trails** — log risk scores alongside deployments

---

## Project Structure

```
commit-risk-predictor/
├── data/                   # Raw and processed commit data
├── models/                 # Saved classifiers and embeddings
├── src/
│   ├── extractor.py        # Diff parsing and hunk extraction
│   ├── embedder.py         # CodeBERT embedding pipeline
│   ├── classifier.py       # Risk scoring model
│   ├── predictor.py        # End-to-end inference
│   └── hook.py             # Git hook integration
├── train.py                # Model training script
├── evaluate.py             # Evaluation and metrics
├── predict.py              # CLI prediction tool
├── setup_hook.py           # Hook installer
└── requirements.txt
```

---

## Tags

`machine-learning` `mlops` `devtools` `codebert` `git` `embeddings` `risk-scoring` `ci-cd` `python` `transformers`

---

## Author

**Sahojit Karmakar**
[GitHub](https://github.com/Sahojit) · [LinkedIn](https://linkedin.com/in/sahojit-karmakar-38972b28a) · [Email](mailto:sahojitxd26@gmail.com)
