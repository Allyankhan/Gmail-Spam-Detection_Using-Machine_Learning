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
- [Email Processing Flow](#email-processing-flow)
- [ML Pipeline](#ml-pipeline)
- [Threat Intelligence Layer](#threat-intelligence-layer)
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

The system is organised into four layers. The Streamlit dashboard sits at the top, driven by `app.py` which coordinates the Gmail API, ML pipeline, and VirusTotal threat intel underneath.

```mermaid
graph TD
    A[" Streamlit Dashboard\nLive results · risk scores · threat flags"]

    B[" app.py — Orchestration\nCoordinates all modules end-to-end"]

    C[" Gmail API\ngmail_api.py\n\nOAuth 2.0 auth\nFetch unread emails\nExtract text · URLs · attachments"]

    D["ML Pipeline\nmodel_handler.py\n\nText preprocessing\nTF-IDF vectorization\nNaive Bayes classifier"]

    E[" Threat Intelligence\nvt_api.py\n\nVirusTotal API v3\nURL scanning · attachment hashing\nMulti-engine verdict"]

    A <-->|"predictions & scores"| B
    B -->|"raw emails"| C
    B -->|"email text"| D
    C -->|"URLs & attachments"| E
    D -->|"spam score"| B
    E -->|"threat verdict"| B

    style A fill:#7c3aed,color:#ede9fe,stroke:#5b21b6
    style B fill:#374151,color:#f9fafb,stroke:#1f2937
    style C fill:#0f766e,color:#ccfbf1,stroke:#0d9488
    style D fill:#1d4ed8,color:#dbeafe,stroke:#1e40af
    style E fill:#b91c1c,color:#fee2e2,stroke:#991b1b
```

---

## Email Processing Flow

Each email passes through two parallel tracks — text classification and threat scanning — before being aggregated into a final risk score and displayed on the dashboard.

```mermaid
flowchart TD
    A([User authenticates\nOAuth 2.0]) --> B[Fetch unread emails\nGmail API]
    B --> C[Extract subject + body text]
    C --> D{Parse email content}

    D -->|Text content| E[Preprocess text\nlowercase · strip HTML · tokenize]
    D -->|URLs found| F[Submit URLs\nto VirusTotal]
    D -->|Attachments found| G[Hash attachments\nSHA-256 → VirusTotal]

    E --> H[TF-IDF vectorize]
    H --> I[Naive Bayes classifier\nP spam = 0.0 – 1.0]

    F --> J[URL threat verdict\nengines flagged / total]
    G --> K[File threat verdict\nantivirus scan results]

    I --> L[Aggregate risk score\nML score + VT flags]
    J --> L
    K --> L

    L --> M{Risk level}
    M -->|Low| N[" Clean\nno action"]
    M -->|Medium| O[" Suspicious\nflagged for review"]
    M -->|High| P[" Malicious\nblocked + alerted"]

    N --> Q[Render on Streamlit dashboard]
    O --> Q
    P --> Q
```

---

## ML Pipeline

The ML pipeline follows a standard scikit-learn pattern: raw text is preprocessed, vectorized with TF-IDF, and scored by a Multinomial Naive Bayes model. Both artefacts are serialised to disk so the app loads them at startup without retraining.

```mermaid
flowchart LR
    A([Raw email text\nsubject + body]) --> B

    subgraph B [" Preprocessing "]
        direction TB
        B1[Lowercase] --> B2[Strip HTML tags]
        B2 --> B3[Remove special chars]
        B3 --> B4[Tokenize + stopword removal]
    end

    B --> C

    subgraph C [" TF-IDF Vectorizer — vectorizer.pkl "]
        direction TB
        C1[Term Frequency per doc] --> C2[Inverse Document Frequency]
        C2 --> C3[Sparse matrix output\nmax 10,000 features · bigrams]
    end

    C --> D

    subgraph D [" Multinomial Naive Bayes — model.pkl "]
        direction TB
        D1["Compute P(spam | features)"] --> D2[Output probability score\n0.0 – 1.0]
        D2 --> D3{Threshold = 0.5}
    end

    D3 -->|score < 0.5| E[" HAM\nlegitimate email"]
    D3 -->|score ≥ 0.5| F[" SPAM\ntrigger threat scan"]
```

### Model parameters

| Parameter | Value |
|---|---|
| Algorithm | Multinomial Naive Bayes |
| Vectorization | TF-IDF |
| Feature type | Email subject + body text |
| Max features | 10,000 |
| N-gram range | (1, 2) — unigrams + bigrams |
| Output | Probability score (0.0 – 1.0) |
| Default threshold | 0.5 |

---

## Threat Intelligence Layer

URLs and attachments found in every email — regardless of ML classification — are submitted to the VirusTotal API. Results from 70+ antivirus engines are aggregated into a single risk verdict.

```mermaid
flowchart TD
    A[Email parsed by gmail_api.py] --> B{Content type}

    B -->|URLs extracted| C["POST /url/scan\nVirusTotal API v3"]
    B -->|File attachments| D["SHA-256 hash\nPOST /file/scan\nVirusTotal API v3"]

    C --> E["Engine results\ne.g. 3 / 72 flagged"]
    D --> F["Antivirus results\nper-engine verdict"]

    E --> G{Positives count}
    F --> G

    G -->|0 positives| H[" Clean"]
    G -->|1–3 positives| I[" Suspicious"]
    G -->|4+ positives| J[" Malicious"]

    H --> K[Return verdict to app.py]
    I --> K
    J --> K
```

---

## Tech Stack

| Component | Technology |
|---|---|
| Language | Python 3.9+ |
| Web UI | Streamlit |
| ML library | scikit-learn |
| Email access | Gmail API (Google Cloud) |
| Threat intel | VirusTotal Public API v3 |
| Auth | OAuth 2.0 (`google-auth`) |
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
