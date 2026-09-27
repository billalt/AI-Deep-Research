---
title: deep_research
app_file: app.py
sdk: gradio
sdk_version: 6.14.0
---

# AI Deep Research

A Python research assistant with a Gradio web interface. Enter a research question and the app uses the OpenAI Agents SDK to plan web searches, gather findings, write a detailed Markdown report, and deliver it by email or Pushover.

## How it works

1. **Plan:** The planner agent generates search queries and a reason for each one. By default, it is instructed to produce five searches.
2. **Search:** Search agents run concurrently using OpenAI's `WebSearchTool`, with instructions to summarize each search in two or three paragraphs under 300 words.
3. **Write:** The writer agent combines the original question and search summaries into a structured result containing a short summary, a Markdown report, and follow-up questions. Its prompt targets at least 1,000 words and roughly 5–10 pages; these lengths are not validated.
4. **Deliver:** A delivery agent prepares a subject, plain-text body, and HTML body, then calls a tool to send the report through SMTP or Pushover.
5. **Display:** The UI shows progress messages followed by the Markdown report. The separate short summary and follow-up questions are not displayed.

Each run also generates an OpenAI trace link, shown in the initial progress update.

## Run locally

You need Python with `venv` and `pip`, an OpenAI API key with access to the configured model and web search tool, and credentials for either SMTP email or Pushover. The repository does not pin a Python version.

```bash
git clone https://github.com/billalt/AI-Deep-Research.git
cd AI-Deep-Research
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

On Windows, activate the environment with `.venv\Scripts\activate` instead.

Create a `.env` file in the project root. For the default email delivery mode:

```dotenv
OPENAI_API_KEY=your-openai-api-key
DEFAULT_MODEL_NAME=gpt-5.4-mini
HOW_MANY_SEARCHES=5

USE_EMAIL=true
EMAIL_ADDRESS=you@example.com
EMAIL_SMTP_SERVER=smtp.example.com
EMAIL_APP_PASSWORD=your-smtp-password
```

Replace the placeholders with your provider's settings. Email is sent **from and to `EMAIL_ADDRESS`** using SMTP on port **587** with STARTTLS. The sender and recipient cannot currently be configured separately.

Keep `.env` out of version control; the repository currently has no root `.gitignore`.

Start the app:

```bash
python app.py
```

Open the local URL printed by Gradio, enter a question or select an example, and click **Investigate** or press Enter. Progress messages update in place until the final report is displayed.

## Configuration

The app loads `.env` with `override=True`, so values in that file override existing environment variables. Restart the app after changing configuration.

| Variable | Default | Purpose |
| --- | --- | --- |
| `OPENAI_API_KEY` | None | API key used by the OpenAI Agents SDK. |
| `DEFAULT_MODEL_NAME` | `gpt-5.4-mini` | Model used by all four agents. |
| `HOW_MANY_SEARCHES` | `5` | Number of searches requested in the planner prompt; must parse as an integer. The actual search count depends on the planner's output. |
| `USE_EMAIL` | `true` | Selects SMTP when the value is `true` (case-insensitive); any other value selects Pushover. |
| `EMAIL_ADDRESS` | None | SMTP login, sender, and recipient in email mode. |
| `EMAIL_SMTP_SERVER` | None | SMTP hostname in email mode. |
| `EMAIL_APP_PASSWORD` | None | SMTP authentication password in email mode. |
| `PUSHOVER_USER` | None | Pushover user key in push mode. |
| `PUSHOVER_TOKEN` | None | Pushover application token in push mode. |

To use Pushover instead of email, keep the OpenAI settings and configure:

```dotenv
USE_EMAIL=false
PUSHOVER_USER=your-pushover-user-key
PUSHOVER_TOKEN=your-pushover-application-token
```

This sends the generated subject and plain-text body to Pushover. Setting `USE_EMAIL=false` switches delivery channels; it does not disable delivery. UI status messages still refer to email in this mode.

## Project structure

| File | Responsibility |
| --- | --- |
| `app.py` | Main Gradio interface and streaming progress updates. |
| `styles.py` | Custom CSS, JavaScript, header markup, and example questions. |
| `research_manager.py` | Coordinates tracing, planning, concurrent searches, report writing, and delivery. |
| `planner_agent.py` | Planner instructions and structured search-plan models. |
| `search_agent.py` | Web search agent and summary instructions. |
| `writer_agent.py` | Report-writing agent and `ReportData` output model. |
| `email_agent.py` | Delivery agent and its email/Pushover function tool. |
| `messenger.py` | SMTP and Pushover transport functions. |
| `simple.py` | Alternative minimal Gradio interface using the same research workflow and an older launch style. |
| `requirements.txt` | Python dependencies: Pydantic, python-dotenv, OpenAI, OpenAI Agents SDK, Gradio, and Requests. |

The README's YAML metadata configures a Gradio Space to launch `app.py` with Gradio `6.14.0`. Local dependencies in `requirements.txt` are unpinned, so local installations may use different versions.

## Current limitations

- Delivery runs before the final report is yielded to the UI. There is no delivery-free mode, and a delivery exception can prevent an already-written report from appearing.
- The orchestration has no application-level retry or recovery handling. A failed search can interrupt the run.
- Pushover requests do not check the HTTP response or split long reports, so the app can report success even if the service rejects the message.
- Reports are generated from search summaries. The prompts do not require source citations or independently verify claims.
- There is no report history, database, or file export, and the repository contains no automated test suite.
