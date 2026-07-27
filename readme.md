# AI-Powered ATS Resume Analyzer
An automated resume screening workflow built in **n8n** that analyzes a candidate's resume against a job description, generates a weighted ATS compatibility score, identifies skill gaps, and emails a recruiter-ready HTML report end to end, with no manual review required.

## Overview

Recruiters manually screening hundreds of resumes is slow and inconsistent. Candidates, meanwhile, have no reliable way to know how well their resume matches a specific job before they apply.

This workflow solves both problems: it accepts a resume (PDF) and a job description via a webhook, runs them through a four-agent AI pipeline, and automatically emails the candidate a structured, professional ATS compatibility report — scored, gap-analyzed, and formatted for readability.

## Architecture

```
Receive Application (Webhook, POST)
   │
   ├──> Extract Resume Text (PDF → text)
   │        └──> Resume Analysis AI
   │                  └──> Prepare Resume Data (+ candidate_email extraction)
   │                                                       │
   └──> Job Analysis AI                                    │
             └──> Prepare Job Data ───────────────────────►│
                                                             ▼
                                          Combine Resume & Job Data (Merge)
                                                             │
                                                             ▼
                                          Get Current Date (inject report_date)
                                                             │
                                                             ▼
                                          ATS Match Analyzer (weighted scoring)
                                                             │
                                                             ▼
                                          Report Formatter AI (recruiter-facing copy)
                                                             │
                                                             ▼
                                          Format ATS Report (Markdown → HTML)
                                                             │
                                                             ▼
                                          Send ATS Report Email (Gmail)
```

---

## Node-by-Node Breakdown

| Node | Type | Purpose |
|---|---|---|
| **Receive Application** | Webhook (POST) | Entry point. Accepts a resume file (`resume`, binary/PDF) and a job description (`job_description`, text) in the request body. |
| **Extract Resume Text** | Extract From File | Converts the uploaded PDF into raw text for downstream parsing. |
| **Resume Analysis AI** | LangChain Agent (Gemini) | Extracts structured candidate data from the raw resume text: name, education, technical skills, programming languages, tools & technologies, projects, certifications, and work experience. Instructed to never invent missing data. |
| **Prepare Resume Data** | Set | Packages the resume analysis into a `resume_analysis` field, and separately extracts the candidate's email address via regex (`candidate_email`) directly from the raw resume text — avoiding reliance on the AI to reliably reproduce it. |
| **Job Analysis AI** | LangChain Agent (Gemini) | Extracts structured job requirements from the job description: required skills, tools & technologies, experience requirements, education requirements, and ATS keywords. |
| **Prepare Job Data** | Set | Packages the job analysis into a `job_description_analysis` field. |
| **Combine Resume & Job Data** | Merge (by position) | Joins the resume and job description branches back into a single item so both analyses are available together. |
| **Get Current Date** | Set | Injects the actual current date (`report_date`, via `$now`) into the pipeline. This prevents the AI from hallucinating a date later on. |
| **ATS Match Analyzer** | LangChain Agent (Gemini) | Compares the resume analysis against the job description analysis and produces the core evaluation: a weighted ATS compatibility score, matched skills, missing skills, a skill-gap analysis, resume improvement recommendations, and interview preparation points. |
| **Report Formatter AI** | LangChain Agent (Gemini) | Rewrites the raw ATS analysis into a clean, concise, recruiter-friendly report, using the injected `report_date` rather than generating its own. |
| **Format ATS Report** | Markdown → HTML | Converts the AI's Markdown-formatted report into proper HTML so it renders correctly in email clients instead of showing raw `**`/`###` symbols. |
| **Send ATS Report Email** | Gmail | Sends the final HTML report to the candidate's extracted email address, with the subject "Your ATS Resume Analysis Report." |


## Scoring Model

The ATS Match Analyzer applies a fixed, weighted rubric:

| Category | Weight |
|---|---|
| Skills Match | 50% |
| Tools & Technologies | 20% |
| Projects & Experience | 20% |
| Education Alignment | 10% |

All agents are explicitly instructed not to invent or assume information beyond what's present in the source documents, and to mark missing data as "Not mentioned" rather than guessing.

## Tech Stack

- **n8n** — workflow orchestration and API entry point
- **Google Gemini** — LLM powering all four AI agents (via `@n8n/n8n-nodes-langchain`)
- **n8n Markdown node** — Markdown-to-HTML conversion for email rendering
- **Gmail (OAuth2)** — automated report delivery
- **Postman** — used for testing the webhook endpoint

## Key Design Decisions

**Four specialized AI agents instead of one large prompt.**
Each agent has a single, narrow responsibility: parse the resume, parse the job description, compare and score, then rewrite for tone. This kept the pipeline easy to debug (each node's output could be inspected independently to isolate failures) and let each stage be reasoned about — and improved — in isolation.

**Email extracted via regex, not AI.**
Early versions asked the AI to return the candidate's email as part of its analysis, which proved unreliable when passed downstream. Extracting it directly from the raw resume text with a regex pattern in **Prepare Resume Data** is deterministic and removes an entire class of failure.

**Date injected via expression, not generated by the AI.**
Letting the AI infer "today's date" caused it to hallucinate an incorrect one. **Get Current Date** generates the real date with n8n's `$now` expression and passes it explicitly into both the ATS Match Analyzer and Report Formatter AI prompts, which are instructed to use it verbatim.

**Dedicated Markdown-to-HTML conversion step.**
The AI naturally writes in Markdown. Email clients don't render Markdown. Rather than relying on prompt instructions to make the AI avoid Markdown syntax (unreliable), a dedicated **Format ATS Report** node converts the output to real HTML every run, guaranteeing correctly rendered emails.


## How to Test This Workflow

This repository contains an **exported n8n workflow file** (a blueprint), not a live, running service. To actually test it, you need your own n8n instance and your own credentials. The JSON does not include working API keys.

1. **Get an n8n instance**
   - Sign up for [n8n Cloud](https://n8n.io) (free tier available), or
   - Self-host n8n locally via Docker or npm
2. **Import the workflow**
   - In n8n: **Workflows → Import from File**, and select the `.json` file from this repo
   - All nodes and connections will load exactly as designed
3. **Connect your own credentials**
   - Add your own **Google Gemini API key** (free from [Google AI Studio](https://aistudio.google.com)) to each of the four Gemini model nodes
   - Connect your own **Gmail account** via OAuth2 to the `Send ATS Report Email` node
4. **Activate the workflow**
   - This generates a new webhook URL unique to your instance — the original URL in this repo will not work for you
5. **Send a test request**
   - Use Postman (or any HTTP client) to `POST` to your new webhook URL with:
     - `resume` — a PDF file (form-data key must be exactly `resume`)
     - `job_description` — plain text
   - Check your inbox for the generated report

> **Note:** Credentials and webhook URLs are instance-specific by design. This keeps the shared workflow safe to publish without exposing any personal API keys or accounts.

## Setup

1. Import the workflow JSON into n8n.
2. Add Google Gemini credentials to all four `lmChatGoogleGemini` model nodes (`ResumeAnalyzerModel`, `JobAnalyzerModel`, `ATSMAtchAnalyzerModel`, `FormatterModel`).
3. Connect a Gmail account via OAuth2 to **Send ATS Report Email**.
4. Activate the workflow and note the generated webhook URL (`/webhook/job-application`).
5. Send a `POST` request with:
   - `resume` — binary PDF file (form-data key must be exactly `resume`)
   - `job_description` — plain text field

### Example Request (Postman)

- **Method:** `POST`
- **URL:** `<your-n8n-instance>/webhook/job-application`
- **Body:** `form-data`
  - `resume`: [attach PDF file]
  - `job_description`: [paste job description text]

## Sample Output

The candidate receives an HTML email titled **"Your ATS Resume Analysis Report"** containing:
- Overall ATS Compatibility Score (0–100%) with category-by-category breakdown
- Strong Matches
- Missing Skills
- Skill Gap Analysis
- Resume Improvement Recommendations
- Interview Preparation Points

## Possible Future Improvements

- **Structured output parsing** — have ATS Match Analyzer return JSON (score, skill arrays) via a Structured Output Parser, enabling downstream automation like auto-filtering candidates below a threshold, instead of only producing free text.
- **Input validation** — reject requests early if `job_description` is missing or the uploaded file isn't a valid PDF, rather than letting empty data silently flow through the pipeline.
- **Error handling** — add a fallback/error branch so failed AI calls or malformed uploads don't fail silently.
- **Persistence** — save each report to a database or spreadsheet (e.g. Airtable, Google Sheets) for recruiter-side tracking, in addition to emailing the candidate.

## Author

Built by Insharah Irshad
BS Artificial Intelligence
