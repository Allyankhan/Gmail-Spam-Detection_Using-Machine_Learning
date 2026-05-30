# 📧 Gmail Spam Detection Using Machine Learning

> A real-time spam detection and threat intelligence system that connects to your Gmail inbox, classifies emails using machine learning, and scans for malicious URLs and attachments via VirusTotal.

[![Python](https://img.shields.io/badge/Python-3.9+-3776AB?style=flat-square&logo=python&logoColor=white)](https://python.org)
[![Streamlit](https://img.shields.io/badge/Streamlit-1.x-FF4B4B?style=flat-square&logo=streamlit&logoColor=white)](https://streamlit.io)
[![scikit-learn](https://img.shields.io/badge/scikit--learn-1.x-F7931E?style=flat-square&logo=scikit-learn&logoColor=white)](https://scikit-learn.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-22C55E?style=flat-square)](LICENSE)

---

## Table of Contents

- [Overview](#overview)
- [System Architecture](#system-architecture)
- [ML Pipeline](#ml-pipeline)
- [Tech Stack](#tech-stack)
- [Installation](#installation)
- [Configuration](#configuration)
- [Project Structure](#project-structure)
- [Model Details](#model-details)
- [Future Improvements](#future-improvements)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

This project is an end-to-end spam detection and threat intelligence system that integrates directly with Gmail using OAuth 2.0. It combines three domains into one production-ready Python application:

| Domain | What it does |
|---|---|
| **Machine Learning** | Classifies emails as spam/ham using a trained Naive Bayes model |
| **Cybersecurity** | Scans URLs and attachments against the VirusTotal threat database |
| **Cloud Integration** | Connects to Gmail in real time via the official Gmail API |

---

## System Architecture

The system is split into four major layers that work in sequence: authentication, data ingestion, ML classification, and threat analysis.

```
┌─────────────────────────────────────────────────────────┐
│                     Streamlit Dashboard                 │
│         (Live results, risk scores, threat flags)       │
└────────────────────┬────────────────────────────────────┘
                     │
          ┌──────────▼──────────┐
          │   Orchestration     │  app.py
          │   (app.py)          │
          └──┬──────────────┬───┘
             │              │
   ┌─────────▼──┐    ┌──────▼──────────┐
   │ Gmail API  │    │  ML Pipeline    │
   │ gmail_api  │    │ model_handler   │
   │            │    │                 │
   │ OAuth 2.0  │    │ TF-IDF + NB     │
   └─────────┬──┘    └──────┬──────────┘
             │              │
             └──────┬───────┘
                    │
          ┌─────────▼────────┐
          │  Threat Intel    │  vt_api.py
          │  (VirusTotal)    │
          │  URL + file scan │
          └──────────────────┘
```

---

## Email Processing Flow

```
User Login (OAuth 2.0)
        │
        ▼
Fetch Unread Emails (Gmail API)
        │
        ▼
Extract Subject + Body Text
        │
        ▼
  ┌─────┴──────┐
  │            │
  ▼            ▼
Text         URLs &
Preprocess   Attachments
  │            │
  ▼            ▼
TF-IDF      VirusTotal
Vectorize     API Scan
  │            │
  ▼            ▼
Naive Bayes  Threat Score
Classifier   (malicious?)
  │            │
  └─────┬──────┘
        │
        ▼
  Aggregate Risk Score
        │
        ▼
  Display in Dashboard
```

### Step-by-step

1. **OAuth handshake** — The user authenticates once via Google's OAuth 2.0 flow. Credentials are stored locally in `credentials.json` for subsequent runs.
2. **Email fetch** — `gmail_api.py` queries the Gmail API for unread or recent messages and returns raw email objects.
3. **Text preprocessing** — Email subject and body are lowercased, stripped of HTML, and tokenized.
4. **TF-IDF vectorization** — The preprocessed text is transformed using the pre-fitted `vectorizer.pkl`.
5. **Classification** — `model.pkl` (Multinomial Naive Bayes) outputs a spam probability score between 0 and 1.
6. **Threat scanning** — Any URLs or file attachments found in the email are submitted to the VirusTotal API via `vt_api.py`.
7. **Risk aggregation** — ML probability + VirusTotal flags are combined into a final risk level (Low / Medium / High).
8. **Dashboard render** — Streamlit displays the results in real time with colour-coded risk indicators.

---

## ML Pipeline

```
Raw Email Text
      │
      ▼
┌─────────────────────────────────┐
│         Preprocessing           │
│  - Lowercase                    │
│  - Strip HTML tags              │
│  - Remove special characters    │
│  - Tokenize                     │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│       TF-IDF Vectorizer         │
│  - Term Frequency               │
│  - Inverse Document Frequency   │
│  - Sparse matrix output         │
│  (saved as vectorizer.pkl)      │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│   Multinomial Naive Bayes       │
│  - P(spam | features)           │
│  - Probability score: 0.0–1.0   │
│  - Threshold: 0.5 (adjustable)  │
│  (saved as model.pkl)           │
└──────────────┬──────────────────┘
               │
       ┌───────┴────────┐
       ▼                ▼
  Score < 0.5      Score ≥ 0.5
    HAM ✅           SPAM 🚩
```

| Parameter | Value |
|---|---|
| Algorithm | Multinomial Naive Bayes |
| Vectorization | TF-IDF |
| Feature type | Email subject + body text |
| Output | Probability score (0.0 – 1.0) |
| Default threshold | 0.5 |

---

## Threat Intelligence Layer

Each email is additionally scanned by the VirusTotal API:

```
Email parsed
     │
     ├── Extract all URLs ──► Submit to VT /url/scan
     │                              │
     │                              ▼
     │                       positives / total
     │                       (e.g. 3/72 engines flagged)
     │
     └── Extract attachments ──► Submit to VT /file/scan
                                        │
                                        ▼
                                 SHA256 hash lookup
                                 + antivirus results
```

Results are summarised as:

| VT Positives | Risk level |
|---|---|
| 0 | Clean |
| 1–3 | Suspicious |
| 4+ | Malicious |

---

## Tech Stack

| Component | Technology |
|---|---|
| Language | Python 3.9+ |
| Web UI | Streamlit |
| ML library | scikit-learn |
| Email access | Gmail API (Google Cloud) |
| Threat intel | VirusTotal Public API v3 |
| Auth | OAuth 2.0 (google-auth) |
| Serialisation | joblib / pickle |

---

## Installation

### Prerequisites

- Python 3.9 or higher
- A Google Cloud project with the Gmail API enabled
- A VirusTotal account (free tier is sufficient)

### Steps

```bash
# 1. Clone the repository
git clone https://github.com/Allyankhan/Gmail-Spam-Detection_Using-Machine_Learning.git
cd Gmail-Spam-Detection_Using-Machine_Learning

# 2. Create and activate a virtual environment
python -m venv venv

# Linux / macOS
source venv/bin/activate

# Windows
venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Add your credentials (see Configuration below)

# 5. Run the app
streamlit run app.py
```

---

## Configuration

### Gmail API

1. Go to [Google Cloud Console](https://console.cloud.google.com/).
2. Create a new project and enable the **Gmail API**.
3. Create OAuth 2.0 credentials (Desktop app type).
4. Download the JSON file and save it as `credentials.json` in the project root.

### VirusTotal API

1. Sign up at [virustotal.com](https://www.virustotal.com) and copy your API key.
2. Create a `.env` file in the project root:

```env
VT_API_KEY=your_virustotal_api_key_here
```

> **Note:** Never commit `credentials.json` or `.env` to version control. Both are listed in `.gitignore`.

---

## Project Structure

```
Gmail-Spam-Detection_Using-Machine_Learning/
│
├── app.py                # Streamlit entry point; orchestrates all modules
├── gmail_api.py          # Gmail OAuth 2.0 auth + email fetching
├── model_handler.py      # Load model/vectorizer, preprocess text, predict
├── vt_api.py             # VirusTotal URL and file scanning
│
├── model.pkl             # Trained Multinomial Naive Bayes model
├── vectorizer.pkl        # Fitted TF-IDF vectorizer
│
├── credentials.json      # Google OAuth credentials (DO NOT COMMIT)
├── requirements.txt      # Python dependencies
└── README.md
```

### Module responsibilities

**`app.py`** — The Streamlit application. Calls `gmail_api` to fetch emails, passes them through `model_handler` for classification, sends URLs/attachments to `vt_api`, and renders the dashboard.

**`gmail_api.py`** — Handles OAuth 2.0 token storage and refresh, and wraps the Gmail API calls to list and fetch message content.

**`model_handler.py`** — Loads `model.pkl` and `vectorizer.pkl` at startup. Exposes a `predict(text)` function that returns a spam probability float.

**`vt_api.py`** — Wraps VirusTotal's `/url/scan` and `/file/scan` endpoints. Returns a structured dict of engine hits and a normalised risk level.

---

## Model Details

The classifier was trained on a labelled spam/ham dataset using the following pipeline:

```python
from sklearn.pipeline import Pipeline
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB

pipeline = Pipeline([
    ('tfidf', TfidfVectorizer(
        max_features=10_000,
        ngram_range=(1, 2),
        stop_words='english'
    )),
    ('clf', MultinomialNB(alpha=0.1))
])
```

| Metric | Score (validation set) |
|---|---|
| Accuracy | ~98% |
| Precision (spam) | ~97% |
| Recall (spam) | ~96% |
| F1 (spam) | ~96.5% |

> Scores are indicative. Re-train on your own labelled dataset for production use.

---

## Future Improvements

- [ ] Deep learning classifier (LSTM or fine-tuned BERT)
- [ ] Phishing-specific detection model
- [ ] Docker containerisation
- [ ] Cloud deployment (AWS / GCP / Azure)
- [ ] Admin monitoring dashboard with historical trends
- [ ] Automated model retraining pipeline
- [ ] Support for attachments beyond URL scanning (YARA rules, sandbox detonation)
- [ ] Multi-account Gmail support

---

## Contributing

Contributions are welcome!

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-feature`
3. Commit your changes: `git commit -m 'Add my feature'`
4. Push to your branch: `git push origin feature/my-feature`
5. Open a Pull Request

Please open an issue first for significant changes so we can discuss the approach.

---

## License

This project is licensed under the [MIT License](LICENSE).

---

## Support

If you found this project useful, please give it a ⭐ on [GitHub](https://github.com/Allyankhan/Gmail-Spam-Detection_Using-Machine_Learning).
