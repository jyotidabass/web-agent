# 🌐 Web Agent

**Web Agent** is an AI-powered browser automation tool that lets you control a real web browser using natural language. You type what you want done — the AI figures out how to do it, clicking, typing, and scrolling just like a human would.

Built with Streamlit, LangChain, Google Gemini, and the `browser-use` library. Usage data and feedback are stored in Google BigQuery.

---

## 📋 Table of Contents

- [How It Works](#how-it-works)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Local Setup](#local-setup)
- [Docker Setup](#docker-setup)
- [Google Cloud / BigQuery Setup](#google-cloud--bigquery-setup)
- [Environment Variables](#environment-variables)
- [Running the App](#running-the-app)
- [Code Walkthrough](#code-walkthrough)
- [Blog Post](#blog-post)
- [Repo Naming](#repo-naming)
- [Known Issues & Notes](#known-issues--notes)

---

## How It Works

1. User enters a **Google Gemini API key** and a **natural language task** (e.g. *"Go to amazon.com and find the best-rated wireless headphones under $50"*).
2. The app creates a **LangChain agent** backed by **Gemini 1.5 Pro**.
3. The agent uses the **`browser-use`** library to open a real browser (via Playwright) and execute the task autonomously — navigating, clicking, typing, scrolling.
4. Results are displayed in the Streamlit UI.
5. User feedback (thumbs up/down + optional comment) is stored in **BigQuery** for improvement tracking.

---

## Tech Stack

| Component | Library / Service |
|---|---|
| UI | [Streamlit](https://streamlit.io/) |
| AI / LLM | [Google Gemini 1.5 Pro](https://ai.google.dev/) via `langchain-google-genai` |
| Agent framework | [LangChain](https://www.langchain.com/) |
| Browser automation | [`browser-use`](https://github.com/browser-use/browser-use) + Playwright |
| Data storage | [Google BigQuery](https://cloud.google.com/bigquery) |
| GCP Auth | `google-cloud-bigquery`, `google-auth` |
| Feedback widget | `streamlit-feedback` |
| Containerization | Docker |

---

## Project Structure

```
web-agent/
├── app.py                   # Main application — all logic lives here
├── requirements.txt         # Python dependencies
├── Dockerfile               # Container definition
├── service-account-key.json # GCP credentials (DO NOT commit — see setup)
└── README.md
```

The entire app is a single file (`app.py`). There are no submodules or separate backend services.

---

## Prerequisites

- Python **3.11+** (3.11.8 used in Docker)
- A **Google Gemini API key** — get one free from [Google AI Studio](https://aistudio.google.com/)
- A **Google Cloud project** with BigQuery enabled and a service account key (JSON)
- Docker (optional, for containerized deployment)

---

## Local Setup

### 1. Clone the repository

```bash
git clone https://github.com/jyotidabass/web-agent.git
cd web-agent
```

### 2. Install Python dependencies

```bash
pip install -r requirements.txt
```

### 3. Install Playwright browsers

`browser-use` drives a real browser via Playwright. You must install the browser binaries separately:

```bash
playwright install --with-deps
```

> This downloads Chromium (and its system dependencies). It's a one-time step per machine.

### 4. Add your GCP service account key

Place your Google Cloud service account JSON file in the project root and name it:

```
service-account-key.json
```

> ⚠️ **Never commit this file.** Add it to `.gitignore`.

### 5. (Optional) Create a `.env` file

The app calls `load_dotenv()` on startup, so you can store environment variables here if needed:

```
# .env
# Add any env vars here — currently the app reads the GCP key from the JSON file directly
```

---

## Docker Setup

The Dockerfile builds a self-contained image with all dependencies and Playwright browsers pre-installed.

### Build

```bash
docker build -t web-agent .
```

### Run

```bash
docker run -p 8080:8080 \
  -v $(pwd)/service-account-key.json:/app/service-account-key.json \
  web-agent
```

Then open `http://localhost:8080` in your browser.

> The `-v` flag mounts your local service account key into the container without baking credentials into the image.

---

## Google Cloud / BigQuery Setup

The app logs session data and user feedback to BigQuery. You need:

### 1. Create a GCP project and enable BigQuery API

Go to [console.cloud.google.com](https://console.cloud.google.com), create a project, and enable the BigQuery API.

### 2. Create a BigQuery dataset named `web_agent`

```bash
bq mk --dataset your-project-id:web_agent
```

### 3. The app will auto-create two tables on first write:

| Table | Columns |
|---|---|
| `session_data` | `session_id` (STRING), `session_creation_time` (TIMESTAMP) |
| `user_feedback` | `session_id` (STRING), `vote` (INTEGER: 1=👍, 0=👎), `comment` (STRING) |

BigQuery's `WRITE_APPEND` job config creates the table automatically on the first load.

### 4. Create a service account

- In GCP Console → IAM → Service Accounts, create a new service account
- Grant it the **BigQuery Data Editor** and **BigQuery Job User** roles
- Download the JSON key and save it as `service-account-key.json` in the project root

---

## Environment Variables

The app uses `python-dotenv` and reads GCP credentials directly from the JSON key file. There are no required environment variables to set. Optional `.env` usage is supported if you want to add custom config later.

---

## Running the App

```bash
streamlit run app.py
```

Opens at `http://localhost:8501` by default.

Enter your Gemini API key in the top-right input, type a task, and hit **Execute 🚀**.

---

## Code Walkthrough

### Initialization (`app.py` top-level)

```python
load_dotenv()
credentials = service_account.Credentials.from_service_account_file('service-account-key.json')
client = bigquery.Client(credentials=credentials, project=credentials.project_id)
```

BigQuery client is initialized once at module load using the service account key.

---

### Session Management

Each user gets a session ID (timestamp-based) that expires after **10 minutes** of inactivity:

```python
def make_new_session(session_data_df):
    session_id = 'id_' + str(datetime.now())
    ...
    upload_to_bq(session_data_df, 'session_data')  # logs to BigQuery
```

On each Streamlit rerun, the app checks if the session is older than 10 minutes and rotates it if so.

---

### The AI Agent (`perform_search`)

This is the core function — it's `async` because `browser-use` is async:

```python
async def perform_search(query, api_key):
    llm = ChatGoogleGenerativeAI(model="gemini-1.5-pro", temperature=0, google_api_key=api_key)
    agent = Agent(task=query, llm=llm)
    result = await agent.run()
    return result.extracted_content()
```

- Creates a Gemini LLM instance with the user-supplied key
- Hands it to a `browser-use` `Agent` along with the user's task
- `agent.run()` opens a real browser and executes the task autonomously
- `extracted_content()` returns a list of strings — intermediate steps + final answer

Because Streamlit is synchronous, the async call is run via a manually managed event loop:

```python
loop = asyncio.new_event_loop()
asyncio.set_event_loop(loop)
results = loop.run_until_complete(perform_search(query, api_key))
```

---

### Displaying Results

```python
c1.markdown(results[:-1])        # intermediate agent steps
c1.markdown(st.session_state.search_results)  # final result (last item)
```

`results` is a list. All items except the last are intermediate agent outputs; the last item is the final extracted answer.

---

### Feedback Collection

```python
streamlit_feedback(
    feedback_type="thumbs",
    optional_text_label="Please provide extra information",
    on_submit=_submit_feedback,
)
```

`_submit_feedback` maps 👍 → `1` and 👎 → `0`, packages it with the session ID, and appends it to BigQuery.

---

### BigQuery Upload

```python
def upload_to_bq(df, table_name):
    destination_table = client.dataset("web_agent").table(table_name)
    job_config = bigquery.LoadJobConfig(write_disposition="WRITE_APPEND")
    load_job = client.load_table_from_dataframe(df, destination_table, job_config=job_config)
    load_job.result()
```

Simple `WRITE_APPEND` — no upserts or deduplication. The table is created automatically on first write.

---

## Blog Post

### 📖 Build Your Own AI Web Agent in Python: A Step-by-Step Guide

> **Read it on Medium (8 min read):**
> 🔗 https://medium.com/@jyotidabass/build-your-own-ai-web-agent-in-python-a-step-by-step-guide-74be67c5b45f?sk=128722450176e5215182614db7baa0e4

Written by [Jyoti Dabass, Ph.D.](https://medium.com/@jyotidabass), this companion article is a complete beginner-friendly deep dive into how this project was designed and built from scratch — explaining every function, every design decision, and every concept in plain English so that even readers new to AI or Python can fully understand and replicate it.

---

### What the Blog Covers

**Part 1 — The Big Picture**
The article opens with the end-to-end user flow: you enter a Gemini API key and a plain-English task, the AI reasons about what steps are needed, a real browser opens invisibly in the background (powered by Playwright), executes the task autonomously, and returns results to your screen. Feedback you submit gets logged permanently to Google BigQuery in the cloud.

**Part 2 — The Libraries (Your Toolbox)**
Every import in `app.py` is explained in plain language — what `streamlit` does (builds the visual UI without any HTML), why `asyncio` is needed (lets the app wait for a browser without freezing), what `browser_use` actually is (the library that controls a real browser like a human), and the role of each Google Cloud library.

**Part 3 — Connecting to Google BigQuery**
The blog uses an intuitive analogy: the `service-account-key.json` file is your private password to Google's cloud vault. The `bigquery.Client` is the connection that opens the vault door. It's initialized once at app startup and reused for every database operation.

**Part 4 — Session Management**
Each user gets a unique session ID generated from the current timestamp (e.g. `id_2025-03-13 10:45:23.123456`), stored in Streamlit's `session_state`. The app checks on every rerun whether the session is older than 10 minutes — if so, it auto-rotates to a fresh one, keeping each period of activity tracked separately.

**Part 5 — The AI Agent (Heart of the App)**
This is the most detailed section. The blog explains:
- Why `perform_search` is declared `async` — so it can pause while waiting for slow browser operations without blocking Streamlit
- Why `temperature=0` is used — for precise, consistent, factual answers with no randomness
- How the `Agent` from `browser-use` takes a plain-English task + the Gemini LLM and autonomously figures out all required browser steps
- Why a manually managed `asyncio` event loop (`asyncio.new_event_loop()`) is needed to bridge Streamlit's synchronous execution model with the async agent
- How `extracted_content()` returns a list where intermediate agent steps are all items except the last, and the final answer is the last item

**Part 6 — Saving Data to BigQuery**
The `upload_to_bq` helper is walked through line by line — pointing to a specific dataset/table, using `WRITE_APPEND` to add rows without deleting history (like appending to a log file), uploading a pandas DataFrame directly, and using `load_job.result()` to wait for confirmation before proceeding.

**Part 7 — The User Interface**
The Streamlit layout is explained: a two-column header (3:1 ratio) for the title and API key input, the `type="password"` field that hides the key as you type, the Execute button, and the spinner that displays while the agent is working.

**Part 8 — Collecting User Feedback**
The `streamlit-feedback` thumbs widget is covered — how clicking a thumb triggers `_submit_feedback`, which converts 👍 to `1` and 👎 to `0`, bundles it with the session ID and optional comment text, and appends the row to BigQuery permanently.

**Part 9 — Docker Packaging**
The Dockerfile is dissected layer by layer — starting from `python:3.11.8`, installing pip dependencies, pre-installing Playwright browser binaries, copying app code, exposing port 8080, and launching Streamlit. The blog describes it as "packing a suitcase" — each layer adds one thing the app needs to run anywhere.

---

> The blog concludes that in under 200 lines of Python, this project combines five powerful tools — Streamlit, Gemini 1.5 Pro, browser-use + Playwright, BigQuery, and Docker — whose patterns (AI agents, async execution, session tracking, feedback loops) are the building blocks of nearly every modern AI product.

---

## Repo Naming

The current repository is named **`web-agent`** — that's a perfectly fine name! It's clear and descriptive.

However, if you want to rename it, here are some options depending on the direction you're going:

| Name | When to use |
|---|---|
| `web-agent` ✅ | Generic, clean — good for open source |
| `browser-agent` | Emphasizes browser automation more specifically |
| `gemini-browser-agent` | Makes the Gemini dependency explicit |
| `ai-web-pilot` | More branded/catchy for a product |

To rename on GitHub: **Settings → Repository name → Rename**. Then update your local remote:

```bash
git remote set-url origin https://github.com/jyotidabass/NEW-NAME.git
```

---

## Known Issues & Notes

- **`share_app()` dialog** references `model-matrimony.app` and `Model Matrimony` branding — this appears to be copied from a different project. Update or remove if you're publishing this independently.
- **No `.gitignore` included** — make sure to add `service-account-key.json` before pushing to a public repo.
- **`asyncio.new_event_loop()`** in Streamlit can cause issues in some deployment environments (e.g. certain Streamlit Cloud versions). If you hit event loop errors, consider wrapping with `nest_asyncio`.
- **Session ID collision** — IDs are generated from `str(datetime.now())` which includes microseconds but is not UUID-based. Extremely unlikely but not guaranteed unique under high load.
- The `load_lottieurl` function and its call are commented out — safe to remove if not needed.

---

## Contributing

PRs and issues welcome! Built by [Jyoti Dabass](https://github.com/jyotidabass).
