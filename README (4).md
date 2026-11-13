# A private AI agent using n8n and Ollama on Selfhost.dev

Built by Srushti Jain, Developer Project Intern.

A small support-ticket triager. A person pastes a customer email into an n8n form. An AI Agent running a local model through Ollama returns a category, an urgency, a two-line summary and a draft reply. The agent has one tool, a Code tool that looks up an order status, to test whether a local model can make tool calls.

It is one workflow and a test set, not a product. Write-up: [link to Google Doc to be added]

## What is in this repository

| File | What it is |
|---|---|
| `workflow.json` | The n8n workflow, "Private Support Ticket Triager (n8n + Ollama)". No credentials inside. |
| `test_emails.json` | The 10 made-up customer emails (ORD-1001 to ORD-1010) with the correct category and urgency for each. |
| `README.md` | This file. |

## How it works

1. A person opens the n8n form and pastes an email.
2. The **On form submission** trigger passes the field `Customer email` to **Form To Email**.
3. **Triage Agent Ollama** (AI Agent) reads the email. If it contains an order ID it calls **Order Status Tool Ollama**.
4. The **Ollama Chat Model** node sends the request to Open WebUI at `/ollama`, which forwards it to Ollama running `llama3.2:3b`.
5. **Format Result** and **Show Result** display category, urgency, summary and draft reply on the form page.

A second path, **Run Test Set -> Test Emails -> Triage Agent Ollama -> Score Ollama -> Totals Ollama**, runs all 10 emails and counts correct category, correct urgency, tool call success and valid JSON.

## Setup

1. In Selfhost.dev, create a new project and deploy the **Open WebUI + Ollama** template with the server size and location the console suggests (mine: cpx22, 2 vCPU, 4 GB RAM, Falkenstein).
2. In the same project use Add service, then Template, and add **n8n**.
3. Open both addresses and create your accounts straight away. The first visitor becomes the owner.
4. In Open WebUI, pull the model `llama3.2:3b` from the admin settings (Admin Panel > Settings > Models).
5. In Open WebUI create an API key: your name > Settings > Account > API Keys. If the option is missing, enable it in Admin Panel > Settings > General.
6. In n8n create an **Ollama account** credential:
   - Base URL: your Open WebUI address with `/ollama` on the end, no trailing slash, no port.
   - API Key: the Open WebUI key from step 5.
   - Allowed HTTP Request Domains: All.
   - Click Retry. It should say "Connection tested successfully".
7. Import `workflow.json` (workflow menu > Import from file), select the credential in the **Ollama Chat Model** node and check the model is `llama3.2:3b`.
8. Paste the contents of `test_emails.json` into the **Test Emails** node if it is not already there.

### Base URLs I tried

| Base URL | Result |
|---|---|
| `http://localhost:11434` | ECONNREFUSED |
| `http://127.0.0.1:11434` | ECONNREFUSED |
| `http://ollama-api-wbw8xc:11434` | EAI_AGAIN |
| Open WebUI address + `/ollama`, with an Open WebUI API key | Works |

n8n reaches Ollama through the public Open WebUI address, not an internal name. Both services are on the same server, but I did not verify whether those packets leave the server, so I do not claim the traffic stays on a private network.

## Running it

- **Form:** Execute workflow from **On form submission**, open the Test URL, paste a made-up email and submit.
- **Test set:** deactivate **Format Result**, **Show Result** and the Gemini nodes, choose **Run Test Set** from the arrow on the orange button, then open **Totals Ollama** and switch to JSON.

## Results (model `llama3.2:3b`, 12 November 2026)

| Number | Run 1 (original prompt) | Run 2 (tightened prompt) |
|---|---|---|
| Tool call success | 10/10 | 10/10 |
| Category accuracy | 1/10 | 5/10 |
| Urgency accuracy | 1/10 | 5/10 |
| Valid JSON | 4/10 (upper bound, see below) | 10/10 |
| Total time for 10 emails | 2 min 53 s | 3 min 22 s |

Run 2 added one rule to the end of the agent system message:

```
FINAL ANSWER RULE: After using the tool, your final reply must be ONLY the JSON object. It must start with { and end with }. Never write a letter, greeting, or "Dear". Put the customer reply inside the draft_reply key.
```

Only `llama3.2:3b` was measured. The qwen2.5 models and the Gemini Flash baseline are not run. See the write-up for the full table, including what was not measured.

## Known limits

- 10 made-up emails and one small model, so treat the numbers as an indication.
- The second prompt was written after seeing run 1 on the same 10 emails.
- The scorer counts anything that parses as JSON, so run 1 "valid JSON" counts a quoted letter as valid and is an upper bound.
- **Score Ollama** shows an error when the workflow is started from the form, because a single form email has no expected answer. It does not affect the result page.
- Times are whole-run totals divided by 10, not per-email medians.

## Privacy

All emails are made up. This repository contains no API key, password or real customer data. If you build your own, keep your Open WebUI API key out of the workflow file and out of screenshots.

## Tools used

I used Claude (an AI assistant from Anthropic) to help troubleshoot connection errors, suggest the system-prompt rule, interpret screenshots and structure the write-up. Measurements and screenshots are from my own account and runs.
