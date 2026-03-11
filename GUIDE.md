# 📖 GUIDE — Automation Control Tower Agent

A simple walkthrough of what this project does, how to run it, and where it can go next.

---

## What This Project Is

This is a **LangGraph-based AI agent** that simulates an email automation pipeline for students. It takes a list of emails, uses GPT-4o to understand them, and decides what automated action to take for each one — like adding a calendar event or setting a reminder.

If an automation fails, it retries once automatically. If it still fails, it escalates to a **Human-in-the-Loop review app** (built on UiPath Action Center) where a person can step in and resolve it. At the end, the agent generates a short Morning Briefing summarising everything.

This is a **simulated prototype** — automations don't trigger real tools yet, but the agent architecture, LLM reasoning, and HITL escalation are fully functional.

---

## Project Structure

```
.
├── main.py            # The entire agent — all nodes, graph, and logic
├── langgraph.json     # LangGraph configuration
├── agent.mermaid      # Visual flow diagram of the agent
├── input.json         # Sample emails to test with
├── bindings.json      # UiPath tool bindings
├── entry-points.json  # UiPath entry point config
├── uipath.json        # UiPath credentials config
├── pyproject.toml     # Python dependencies
├── AGENTS.md          # Node-level descriptions
└── CLAUDE.md          # LLM prompt config
```

---

## How to Run It

### 1. Install dependencies

```bash
pip install -e .
```

### 2. Add your UiPath credentials

Open `uipath.json` and fill in your tenant details:

```json
{
  "tenantUrl": "https://cloud.uipath.com/YOUR_ORG/YOUR_TENANT",
  "clientId": "YOUR_CLIENT_ID",
  "clientSecret": "YOUR_CLIENT_SECRET"
}
```

> Get these from **UiPath Automation Cloud → Admin → API Access**

### 3. Run

```bash
python main.py
```

---

## Sample Input

The agent expects a list of emails. See `input.json`:

```json
{
  "emails": [
    { "subject": "Mid Semester Exam Schedule Released", "body": "Exams begin Monday." },
    { "subject": "Google STEP Internship Applications Open", "body": "Closes in 3 days." },
    { "subject": "Project Meeting Tomorrow", "body": "Meeting at 4 PM." }
  ]
}
```

---

## What Happens Step by Step

All logic lives in `main.py`. Here's what each node does:

**`FetchInputs`**
Loads the emails into `GraphState` and prints them. Nothing fancy — just the starting point.

**`ExtractTasks`**
Sends all emails to GPT-4o and asks it to return an actionable task for each one as a JSON list.

**`ClassifyIntent`**
Sends the extracted tasks back to GPT-4o and classifies each one into: `meeting_request`, `exam_update`, `internship_opportunity`, `promotion`, or `general`.

**`EvaluatePriority`**
Rule-based — no LLM here. Exam and meeting intents get `high`, internships get `medium`, everything else gets `low`.

**`ToolSelector`**
Maps each intent to a tool name:

| Intent | Tool |
|---|---|
| `meeting_request` | `CreateCalendarEvent` |
| `exam_update` | `AddExamReminder` |
| `internship_opportunity` | `CreateApplicationReminder` |
| `promotion` | `ArchiveEmail` |
| `general` | `NoAction` |

**`ExecuteAutomation`**
Currently **simulated** using `random.random() > 0.3` to decide success or failure. In a real deployment this would call actual UiPath processes.

**`VerifyExecution`**
Checks results. Any failures go into `failed_actions` and the router decides what to do next.

**`RetryAutomation`**
Increments `retry_count` and loops back to `ExecuteAutomation` once. After one retry, failures escalate.

**`HumanApproval`**
Uses `interrupt()` + UiPath `CreateTask()` to pause the agent and create a task in the **UiPath Action Center app**. A human reviewer sees the failed actions, resolves them in the app, and the agent resumes.

**`GenerateBrief`**
Sends all execution results to GPT-4o and generates a plain-English Morning Briefing summary.

---

## The Human-in-the-Loop App

When automations fail after retry, the agent calls:

```python
interrupt(
    CreateTask(
        app_name="Automationapprovalapp",
        title="Automation Requires Manual Review",
        data={"FailedActions": str(state.failed_actions)},
        app_version=1,
        app_folder_path="Sneha"
    )
)
```

This creates a task inside **UiPath Action Center** — a web UI where a reviewer can see what failed and take action. Once they respond, the agent continues and generates the final briefing.

---

## What Could Be Built Next

This prototype lays the groundwork for a production-ready system. Some natural next steps:

- **Real automation triggers** — replace the `random` simulation in `ExecuteAutomation` with actual UiPath process calls via the Orchestrator API
- **Live email connection** — connect `FetchInputs` directly to a Gmail or Outlook inbox using their APIs, so the agent runs automatically on new mail
- **Smarter priority** — let the LLM evaluate urgency from the email body instead of using a fixed rule map (e.g. detect "closes in 24 hours" → High)
- **Persistent memory** — store past emails and actions in a database so the agent builds context over time and avoids duplicate reminders
- **Scheduled runs** — deploy on UiPath and trigger automatically every morning

---

## Dependencies

All listed in `pyproject.toml`. Key ones:

| Package | Purpose |
|---|---|
| `langgraph` | Agent graph and node orchestration |
| `langchain-core` | LLM message handling |
| `uipath` | UiPath SDK and Action Center integration |
| `uipath-langchain` | UiPath-hosted GPT-4o access |
| `pydantic` | Typed `GraphState` |
