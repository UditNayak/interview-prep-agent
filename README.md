# Interview Prep Agent (n8n)

An agentic n8n workflow that turns a panicked "I have an interview in X days" form
into a researched, day-by-day prep plan, delivered to your inbox as a formatted email.

It web-searches real interview experiences, runs four role-scoped AI agents
(intelligence, reality-check, planner, behavioural), enforces the plan's structure
with deterministic code, and emails the result. Optionally it also drops one
calendar event per prep day into Google Calendar.

- **AI** (Groq llama-3.3-70b, with a Gemini 2.5 Flash fallback on every step) does
  the judgement: summarising research, diagnosing readiness, drafting the plan,
  coaching behaviour.
- **Deterministic code** does everything that must be reliable: scoring inputs,
  choosing prep intensity, repairing the plan to the exact day/hour limits, and
  building the email + calendar events.

---

## 1. What you need

| Key | Where to get it | Used for |
|-----|-----------------|----------|
| `GROQ_API_KEY`   | https://console.groq.com/keys     | Primary LLM (llama-3.3-70b) |
| `GEMINI_API_KEY` | https://aistudio.google.com/apikey | Fallback LLM (gemini-2.5-flash) |
| `TAVILY_API_KEY` | https://app.tavily.com            | Web search |
| SMTP login       | your email provider (Gmail works) | Sending the report email |

Plus Docker Desktop (or Docker Engine + Compose).

---

## 2. Start n8n

```bash
cp .env.example .env          # then open .env and paste your keys + FROM_EMAIL
docker compose up -d
```

Open http://localhost:5678 and create your n8n owner account (first run only).

> `FROM_EMAIL` in `.env` is the address the report is sent *from*. For Gmail,
> set it to your Gmail address.

---

## 3. Import the workflow

1. In n8n: top-right menu -> **Import from File**.
2. Choose `workflows/interview-prep-agent.workflow.json`.
3. The canvas loads with 21 nodes. Two of them need credentials (next step).

---

## 4. Add the two credentials (one-time)

API keys flow in from `.env` automatically, but **n8n credentials cannot be
imported from JSON** — you create them once inside n8n and attach them.

### 4a. SMTP credential (for the report email) — required

1. Click the **Send Email** node -> **Credential to connect with** -> **Create New**.
2. Choose **SMTP**.
3. For a Gmail account:
   - **User**: your full Gmail address
   - **Password**: a **Gmail App Password**, *not* your normal password.
     Create one at https://myaccount.google.com/apppasswords
     (requires 2-Step Verification turned on). It looks like `abcd efgh ijkl mnop`.
   - **Host**: `smtp.gmail.com`
   - **Port**: `465`
   - **SSL/TLS**: ON
4. Save. Make sure the **Send Email** node's *From Email* resolves to the same
   address (it reads `FROM_EMAIL` from `.env`).

Other providers: use their SMTP host/port (e.g. Outlook `smtp.office365.com:587`).

### 4b. Google Calendar credential — only if you'll use the toggle

The calendar step is OFF by default (you choose per run on the form). To make it
work when switched on:

1. Click **Google Calendar - Create Events** -> **Create New** credential ->
   **Google Calendar OAuth2 API**.
2. Follow n8n's Google OAuth setup (create OAuth client in Google Cloud Console,
   paste Client ID/Secret, connect). n8n's guide:
   https://docs.n8n.io/integrations/builtin/credentials/google/
3. Until you add this, just pick **No** for the calendar question on the form and
   the workflow ignores the calendar branch entirely.

---

## 5. Run it

1. Open the **Intake Form** node -> copy the **Test URL** (or **Production URL**
   once the workflow is Active).
2. Open that URL, fill the form (company, role, your email, days, hours, your DSA
   and behavioural situation, weak/strong topics, and the calendar toggle).
3. Submit. Within a few seconds the report lands in the inbox you entered, and the
   form shows an "All set" confirmation.

To make the form URL permanent, toggle the workflow **Active** (top-right) and use
the Production URL.

---

## 6. How the run flows (21 nodes)

```
Intake Form
  -> Validate and Normalize            (deterministic: scores, intensity, safe defaults)
  -> Tavily Web Search                 (tool: real interview experiences)
  -> Agent 1 Interview Intelligence    (Groq)  --on error--> Fallback Gemini --> Parse Intelligence
  -> Agent 2 Reality Check             (Groq)  --on error--> Fallback Gemini --> Parse Reality
  -> Agent 3 Roadmap Planner           (Groq)  --on error--> Fallback Gemini --> Validate and Repair Roadmap
  -> Agent 4 Behavioural Prep          (Groq)  --on error--> Fallback Gemini --> Parse Behavioural
  -> Assemble Report                   (deterministic: builds the HTML email)
  -> Send Email                        (emails the report)
  -> Calendar Enabled?  --No--> Done
                        --Yes--> Build Calendar Events -> Google Calendar -> Done
```

Every Groq node has a Gemini fallback wired to its error output, so a Groq
rate-limit or outage degrades to Gemini instead of failing the run. If both fail,
the Parse nodes substitute safe defaults so the email still sends.

See `docs/architecture.md` and `docs/workflow-explanation.md` for the per-node "why".