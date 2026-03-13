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
git clone https://github.com/your-username/web-agent.git
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
git remote set-url origin https://github.com/your-username/NEW-NAME.git
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

PRs and issues welcome! Built by [Bhavishya Pandit](https://www.linkedin.com/in/bhavishya-pandit/).
