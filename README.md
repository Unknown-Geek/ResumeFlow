# ResumeFlow - Autonomous Agentic Resume Assistant
<img width="2710" height="2170" alt="Frame 1321321610" src="https://github.com/user-attachments/assets/bc5a449f-7905-496b-9df3-00896ab5430b" />
<img width="2710" height="2170" alt="Frame 1321321609" src="https://github.com/user-attachments/assets/fe89bb0a-832f-4993-abbe-6833361b5fd9" />

## Overview

Resume Bot is built as an n8n automation workflow. It connects a Telegram interface to a Supabase database and an Ollama-served large language model, orchestrating a full pipeline from raw resume input to compiled PDF output.

The core idea is that most people maintain a single static resume document. This bot replaces that with a living, structured profile stored in a database. You can update it conversationally, view it at any time, and export it as a professionally formatted PDF. When applying for a specific role, you can provide the job description and the bot will generate a temporarily optimized version of your resume without touching your original data.



## Workflow

<img width="1307" height="831" alt="image" src="https://github.com/user-attachments/assets/a8746f6d-0278-455a-a47b-8b0a0b084950" />


## Features

### Resume Parsing
Drop a PDF into the Telegram chat. The bot extracts the text, passes it to an AI agent, and maps the unstructured content into a structured profile schema. The parsed profile is saved to your Supabase database and becomes your permanent Master Profile.

### Conversational Profile Updates
You do not need commands to update your profile. Send a plain message like "I just completed a machine learning course" or "Add my new frontend internship at Acme Corp" and the AI agent will detect the new information, update the relevant fields in your profile, and confirm what changed. The bot also automatically refreshes your professional summary on every successful update to ensure all facts stay coherent.

### Conversational Profile Queries
Ask the bot about your saved profile data at any time. If you want to know what your Semester 1 score was, or whether you have a specific skill listed on your resume, simply ask. The AI agent will query your stored profile and provide a direct answer without modifying your database records.

### Job Description Optimization
Send `/optimize` followed by a full job description. The bot performs a gap analysis between your master profile and the requirements of the role, rewrites your summary and bullet points to emphasize relevant experience, reorders your skills to prioritize JD-relevant categories, and saves the result as a temporary Tailored Profile. Your master profile is never modified during this process.

### ATS Keyword Gap Analysis
Send `/analyze` followed by a full job description. The bot calculates an ATS match percentage and returns a structured gap analysis highlighting missing hard skills, soft skills, keywords, and specific recommendations on how to improve your score. 

### PDF Export
Request a PDF at any time. The bot generates a LaTeX document from your profile, compiles it on a remote server via SSH, and sends you the resulting PDF. 
- **Double Compilation Pass**: The compilation process automatically runs `pdflatex` twice. The first pass generates cross-reference data; the second resolves `hyperref` links, page references, and page numbers correctly.
- **Auto-Cleanup**: Once the compiled PDF binary is captured, the server immediately cleans up the `.tex`, `.pdf`, `.log`, and `.aux` files from `/tmp` to prevent disk storage accumulation.

### Cover Letter Generation
Send `/coverletter` after running `/optimize`. The bot dynamically pulls your tailored profile (including your tailored tagline and header) to generate a structured, professional cover letter, compiles it as a PDF, and sends it directly to the chat.

### Profile Preview (Segmented Messaging)
View your full profile in formatted text directly in the Telegram chat before exporting, for both the master and tailored versions.
- **Smart Button Attachment**: If your profile exceeds Telegram's 4,096-character limit, the preview is safely split into multiple message segments. The bot dynamically tags the final chunk and attaches the "Download PDF" inline keyboard button **only to the last segment**, providing a clean and professional user interface.
- **Clean Chat Previews**: The bot automatically disables Telegram's web page link previews across all text messages. This prevents massive image thumbnails (like GitHub profile pictures) from cluttering the chat when displaying your URLs. It also intelligently formats links as `Display Text (URL)` when both are present.

### Smart Link Extraction
The bot utilizes robust Regex to automatically detect naked domains (e.g., `techsprouts.netlify.app`, `github.com`) even if you forget to include the `http://` protocol. It dynamically injects `https://` under the hood before compiling the LaTeX PDF to prevent broken hyper-references, ensuring all contact links, live demos, and GitHub repositories are perfectly clickable.


## Commands

| Command | Description |
|---|---|
| `/start` | Display the help menu and command list |
| `/view` | Show your master profile as formatted text in chat (PDF download button at the bottom) |
| `/download` | Compile and download your master profile as a PDF |
| `/optimize [job description]` | Analyze the JD and generate a tailored profile |
| `/viewOptimized` | Show your current tailored profile as formatted text (PDF download button at the bottom) |
| `/downloadOptimized` | Compile and download your tailored resume as a PDF |
| `/coverletter` | Generate a cover letter PDF. Optionally append a job description: `/coverletter [paste JD]` |
| `/analyze [job description]` | Run an ATS keyword gap analysis against a job description |
| `/clear` | ⚠️ Permanently delete your entire profile from the database. The bot will prompt you to start fresh. |

Any other message (including a PDF upload) is treated as a conversational update to your profile. All commands are fully case-insensitive (e.g. `/DownloadOptimized` works identically).


## Architecture

The bot is implemented as a single n8n workflow exported as `ResumeBot.json`. The workflow consists of the following major components:

### Trigger and Routing
- **Telegram Trigger**: Listens for incoming messages and callback queries (inline button presses).
- **Double-Lookup Fetch**: 
  1. **Supabase Fetch by ID**: Looks up the profile by the incoming Telegram chat ID (`telegram_id`).
  2. **Supabase Fetch by Username**: If the user is not found by ID (e.g. they changed accounts/cleared history), it queries the database using their unique Telegram username (`telegram_username`).
  3. **Supabase Fetch (Code Node)**: Merges the results of both searches. If found by username, the Code node returns the old row, ensuring the very first execution has access to their complete profile.
  4. **Check Migration Needed & Supabase Migrate ID**: If found by username but not by ID, a migration is triggered. The bot automatically updates that row's `telegram_id` to the new ID in Supabase. All future fetches will immediately resolve by ID!
- **Command Router (Case-Insensitive)**: A Switch node that inspects the incoming message or callback data and routes execution gracefully regardless of user casing.
  - **Early Evaluation Route**: The router prioritizes `info` (`/start`), `greeting`, and `profileIncomplete` branches *first*.
  - **Greeting Interceptor**: Standard conversational greetings (e.g., "hi", "hello") are intercepted early and handled with direct friendly responses instead of trigger-heavy profile warnings.
  - **Sufficiency Gatekeeper**: No command works if the user's master profile is empty. If data is insufficient, the bot prompts the user specifically for the missing details (e.g., education, work history) and suppresses inline keyboard export buttons until a complete master profile is ingested. The `/clear` and `/view` commands are explicitly exempt from this gate and always execute regardless of profile state.
- **Heuristic-First Intent Classifier**: Before attempting a database update, the bot uses a fast Regex heuristic to check if the user is asking a question (e.g. "what is my GPA?"). If ambiguous, it falls back to an AI Classifier Model to definitively route the message to either the Query branch or the Update branch.

### Profile Branches
- **PDF Upload Branch**: Detects when a document is attached, extracts PDF text, formats a prompt, and sends it to the AI Agent for parsing.
- **Chat Branch**: Handles all plain text messages. Sends them directly to the AI Agent for conversational profile updating.
- **Info Branch**: Returns the help message with an interactive inline command menu.
- **View Branch**: Formats and returns the master profile as Telegram-formatted text.

### AI Agents (Ollama)
All AI inference runs locally via Ollama using the `deepseek-v4-flash` model served through an OpenAI-compatible API endpoint.

- **Intent Classifier Agent**: Evaluates user messages to explicitly route between database updates ("UPDATE") and read-only profile questions ("QUERY").
- **AI Agent** (main): Handles both PDF parsing and conversational updates. Given the current profile and the user's message, it returns a JSON object containing an updated `master_profile` and a natural language `chat_reply`.
- **Query Agent**: Answers user questions directly by reading the stored profile without modifying any data.
- **AI JD Agent**: Receives the user's master profile and a job description. Returns a `tailored_profile` with rewritten content and a `chat_reply` summarizing optimizations and skill gaps.
- **ATS Gap Agent**: Analyzes a job description against the master profile to output a plain-text breakdown of missing hard/soft skills and actionable recommendations.
- **AI Fix LaTeX Agent**: When `pdflatex` compilation fails, this agent receives the compile log and the original LaTeX source, fixes any errors, and returns corrected LaTeX.
- **AI Cover Letter Agent**: Receives the tailored profile and generates a structured cover letter as a JSON object with distinct paragraphs and bullet points.

### Database (Supabase)
A Supabase `Profiles` table stores one row per user, keyed by `telegram_id`. Each row holds:
- `telegram_id`: The user's Telegram chat ID (used as the primary key for lookups)
- `telegram_username`: The user's unique Telegram username (used as a fallback lookup for account migration)
- `master_profile`: JSON blob containing the full structured resume profile
- `tailored_profile`: JSON blob containing the most recent JD-optimized profile

The bot performs an upsert pattern: it checks whether a row exists for the user and issues either an UPDATE or an INSERT accordingly.

### LaTeX and PDF Compilation
Profile data is converted to a LaTeX document using a custom template with precise resume formatting. The `.tex` file is sent to a remote Linux server via SSH, compiled with `pdflatex`, and the resulting PDF is base64-encoded and returned to n8n. The PDF binary is then sent to the user via the Telegram `sendDocument` API.


## Data Model

The `master_profile` and `tailored_profile` fields follow this JSON schema:

```json
{
  "name": "string",
  "tagline": "string",
  "contact": {
    "email": "string",
    "phone": "string",
    "linkedin_url": "string",
    "linkedin_display": "string",
    "github_url": "string",
    "github_display": "string",
    "portfolio_url": "string",
    "portfolio_display": "string"
  },
  "summary": "string",
  "skills": [
    { "category": "string", "items": "string" }
  ],
  "work_experience": [
    {
      "title": "string",
      "company": "string",
      "location": "string",
      "start_date": "string",
      "end_date": "string",
      "bullet_points": ["string"]
    }
  ],
  "education": [
    {
      "degree": "string",
      "institution": "string",
      "score": "string",
      "year": "string"
    }
  ],
  "projects": [
    {
      "name": "string",
      "tech_stack": "string",
      "duration": "string",
      "link_url": "string",
      "link_display": "string",
      "bullet_points": ["string"]
    }
  ],
  "certifications": [
    {
      "name": "string",
      "issuer": "string",
      "year": "string"
    }
  ]
}
```


## How the Cover Letter Command Works

The `/coverletter` command supports two modes depending on whether a job description is provided.

**Without a JD (`/coverletter`):**
The bot dynamically builds the LaTeX cover letter using your current `tailored_profile` (including pulling the optimized tagline for the header). If no tailored profile exists yet, it falls back to your master profile. This is suitable for sending a general application aligned with your existing profile.

**With a JD (`/coverletter [paste full JD here]`):**
The bot uses your `master_profile` combined with the provided job description. The AI writer receives both the profile and the JD in its context, allowing it to write a cover letter targeted to that specific role without requiring a prior `/optimize` run. 

In both modes the output is a compiled PDF sent directly to the chat, using the exact same LaTeX header styling as the resume.


## Humanizer (Anti-AI Writing Style Rules)

To prevent generated resumes, profile summaries, experience descriptions, and cover letters from sounding artificial, formulaic, or obviously machine-generated, all generative LLM agents adhere to strict **Humanizer Constraints**. Additionally, emojis are globally banned across all LLM-generated profile content and chat replies.

These rules mathematically filter out the primary indicators of LLM-generated text:

### 1. Hard Punctuation Constraints
- **Em Dash & Double Hyphen Removal**: The agents are strictly prohibited from generating em dashes (`—`), en dashes (`–`), or double hyphens (`--`/`---`) within text blocks. They must structure sentences naturally using standard periods, commas, colons, or parentheses.

### 2. Copula Restoration (Avoiding AI Verb Stretches)
- **Use Simple Copulas**: The model is forbidden from using complex LLM verb replacements like *"serves as"*, *"stands as"*, *"boasts"*, *"features"*, *"offers"*, or *"marks"*. Instead, simple, direct verbs like *"is"*, *"are"*, *"has"*, or direct action verbs must be used.

### 3. AI-Code Vocabulary Blacklist
- The agents cannot use typical high-frequency AI buzzwords, including: *delve, testament, beacon, synergy, seamless, cutting-edge, revolutionary, leverage, leveraging, transformative, meticulously, expertly, proven track record, elevate, tapestry, foster, multifaceted, catalyst, passionate, dynamically, align with, crucial, enduring, enhance, garner, highlight (verb), interplay, intricate, intricacies, key (adjective), landscape (abstract), pivotal, showcase, underscore.*

### 4. Grammar and Structure Refinement
- **No Present Participle Padding**: The model cannot tack on superficial present-participle `"-ing"` endings (e.g. *"...highlighting their expertise"*) simply to pad bullet points.
- **Rule of Three Prevention**: Restricts forcing bullet points or lists into rigid symmetrical groups of three.
- **No Negative Parallelisms**: Avoids repetitive *"Not only... but also..."* constructs.
- **Active Voice**: Strongly favors direct active voice over passive voice.

### 5. Genuine Voice & Style
- **Grounded, Understated Tone**: Banish sycophantic chatbot filler (*"Certainly!"*, *"I hope this helps!"*) and template cover letter openings in favor of a calm, grounded statement of interest.


## Prerequisites

Before setting up and importing this workflow, ensure the following are available and configured:

- **n8n** (self-hosted or cloud) — version compatible with `typeVersion` 1.x nodes and the LangChain integration nodes
- **Telegram Bot** — created via BotFather; you will need the bot token
- **Supabase project** — with a `Profiles` table containing columns: `telegram_id` (text, primary key), `telegram_username` (text), `master_profile` (jsonb), `tailored_profile` (jsonb)
- **Ollama** — running locally or on a server accessible from n8n, with the `deepseek-v4-flash` model pulled. Ollama must be accessible via an OpenAI-compatible API endpoint (default: `http://localhost:11434/v1`)
- **Remote Linux server with pdflatex** — accessible via SSH with password authentication. Must have `texlive-full` or equivalent packages installed, including `latexmk`, `pdflatex`, and all packages used in the template


## Setup

### 1. Prepare Supabase

Create a new Supabase project and run the following SQL to create the profiles table:

```sql
CREATE TABLE "Profiles" (
  telegram_id TEXT PRIMARY KEY,
  telegram_username TEXT,
  master_profile JSONB,
  tailored_profile JSONB
);
```

If you have an existing database, run this migration command to add the username column:

```sql
ALTER TABLE "Profiles" ADD COLUMN telegram_username TEXT;
```

### 2. Set Up Ollama

Install Ollama on your server and pull the required model:

```bash
ollama pull deepseek-v4-flash
```

Ensure the Ollama API is accessible from your n8n instance. If Ollama and n8n are on different hosts, expose the API with:

```bash
OLLAMA_HOST=0.0.0.0 ollama serve
```

### 3. Set Up the PDF Compilation Server

On a Linux server, install the required LaTeX packages:

```bash
sudo apt-get install texlive-full
```

Ensure SSH password authentication is enabled and the n8n server can reach this host.

### 4. Configure n8n Credentials

In your n8n instance, create the following credentials:

**Telegram API** (name: `AI Agent Bot`)
- Credential type: Telegram API
- Access Token: your bot token from BotFather

**Supabase API** (name: `Resume Bot`)
- Credential type: Supabase API
- Host: your Supabase project URL
- Service Role Secret: your Supabase service role key

**OpenAI API** (name: `Ollama API`)
- Credential type: OpenAI API
- API Key: any non-empty string (Ollama does not require authentication)
- Base URL: your Ollama server's OpenAI-compatible endpoint (e.g., `http://localhost:11434/v1`)

**SSH Password** (name: `SSH Password account`)
- Credential type: SSH
- Host: your LaTeX compilation server's IP or hostname
- Port: 22 (or your configured SSH port)
- Username: the SSH user
- Password: the SSH password

### 5. Import the Workflow

1. Open your n8n instance and navigate to Workflows.
2. Click "Import from file" and select `ResumeBot.json`.
3. After import, open each credential reference in the nodes and map them to the credentials created in step 4. The credential IDs embedded in the JSON are environment-specific and will need to be re-linked.
4. Activate the workflow.

### 6. Configure the Telegram Webhook

n8n automatically registers the webhook with Telegram when the workflow is activated. Ensure:
- n8n is accessible via HTTPS on a public URL
- The Telegram Trigger node has the correct bot token credential


## Error Handling

- **Early Exit Protections**: If a user issues an optimization or analysis command without a JD attached (or without an active profile), the bot executes *early exit rules* to immediately intercept the request, bypassing the heavy generation nodes to quickly return helpful usage warnings while hiding any inline keyboard action buttons.
- **LaTeX Compilation Failure (Auto-Repair)**: If `pdflatex` returns a non-zero exit code, the workflow captures the compile log and invokes the AI Fix LaTeX Agent to identify and correct the error. The corrected LaTeX is passed to the **`Base64 Encode Fixed LaTeX`** node, which base64-encodes the content to protect it from terminal shell expansion/corruption before being written to the SSH server for a retry compile. If the retry fails, the user is notified.
- **Missing or Incomplete Profile Gate**: If a user attempts to run commands without a sufficiently populated master profile, the bot intercepts the execution early, prompts them for the missing details, and suppresses download buttons.
- **JSON Parse Failures**: AI agent outputs are parsed defensively. If the output cannot be parsed as JSON, a fallback response is returned to the user.
