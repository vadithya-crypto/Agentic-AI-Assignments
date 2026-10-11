# Assignment 4 – Transaction Risk Investigation: Multi-Agent System (n8n)

## Sequential Multi-Agent Architecture using n8n, Ollama and Tavily

## 1. Problem Statement

When a bank's monitoring system flags suspicious activity, an investigator has to do three different jobs:

1. **Understand the case:** pull out the facts and spot the red flags.
2. **Research:** check which known fraud or money-laundering scheme the case resembles and whether regulators or news have reported similar cases recently.
3. **Decide:** score the risk, choose an action (clear, monitor, escalate, block and report), and write a case note.

A single LLM prompt doing all three tends to skip steps and mix facts with assumptions. This assignment splits the investigation across **three specialised AI agents** that work in sequence, each passing its output to the next, the way an L1 analyst, a researcher and a decision-maker would on a real fraud team.

## 2. Objectives

- Design and implement a **sequential Multi-Agent System (MAS)** in n8n.
- Give each agent one clear role, its own system prompt and its own output format.
- Hand off context explicitly: each agent receives the original case **and** the earlier agents' outputs.
- Give the research agent a **real tool** (Tavily web search) to bring in current external evidence with source URLs.
- Add conversational memory so the investigator can ask what-if follow-ups.
- Run every agent on a **local LLM (Ollama)** so case data does not leave the machine.

## 3. Architecture

```text
                          USER (investigator)
                                  │  describes the case
                                  ▼
                    ┌──────────────────────────┐
                    │  When chat message       │
                    │  received                │
                    └────────────┬─────────────┘
                                 ▼
  Simple Memory ──► ┌──────────────────────────┐
                    │ AGENT 1                  │
                    │ Case Intake & Red Flags  │   facts, missing info, red flags,
                    └────────────┬─────────────┘   candidate typologies, search keywords
                                 ▼
  Tavily Search ──► ┌──────────────────────────┐
                    │ AGENT 2                  │
                    │ Typology Research        │   web-backed findings + source URLs
                    └────────────┬─────────────┘
                                 ▼
                    ┌──────────────────────────┐
                    │ AGENT 3                  │
                    │ Risk Decision            │   score, decision, actions, case note
                    └────────────┬─────────────┘
                                 ▼
                          FINAL ASSESSMENT (chat)

  All three agents use: Ollama Chat Model – llama3.1:latest (temperature 0.2)
```

**Pattern:** sequential (pipeline) multi-agent system. Control passes in a fixed order; each agent's output becomes the next agent's input.

## 4. The Agents

| Agent | Role | Input | Tool | Output |
|---|---|---|---|---|
| **1 – Case Intake** | Structures the case, lists red flags, proposes candidate typologies and search keywords. Rebuilds the full case on follow-ups. | User's message + conversation history | Simple Memory | Case Facts, Missing Information, Red Flags, Candidate Typologies, Search Keywords |
| **2 – Typology Research** | Checks the candidate typologies against current web evidence and Indian advisories. | Original case + Agent 1 output | **Tavily Search** | Typology Findings, Recent Context with URLs, Most Likely Typology |
| **3 – Risk Decision** | Scores the risk and decides the action; writes the case note. Final voice to the user. | Original case + Agent 1 + Agent 2 outputs | – | Risk score and band, Decision, Immediate Actions, Information to Collect, Case Note, Sources |

### Handoff expressions used in n8n
- Agent 2 prompt: `{{ $('When chat message received').item.json.chatInput }}` and `{{ $json.output }}` (Agent 1's output)
- Agent 3 prompt: the original case, `{{ $('Agent 1 - Case Intake').item.json.output }}` and `{{ $json.output }}` (Agent 2's output)

### Decision scale used by Agent 3
| Score | Band | Typical decision |
|---|---|---|
| 0–30 | Low | CLEAR |
| 31–60 | Medium | MONITOR for 30 days |
| 61–85 | High | ESCALATE to L2 investigator |
| 86–100 | Critical | BLOCK & REPORT (freeze debits, Principal Officer, consider STR to FIU-IND) |

The full system prompts are in each agent node and in `Prompts.md`.

## 5. Tools and Models

| Component | Choice |
|---|---|
| Orchestration | n8n (self-hosted) |
| **LLM (all agents)** | **`llama3.1:latest`** (Llama 3.1 8B) via Ollama, temperature 0.2 |
| Web search tool | Tavily (community node `@tavily/n8n-nodes-tavily`) |
| Memory | Simple Memory (window of 5) on Agent 1 |
| Interface | n8n Chat |

## 6. Setup and Run

1. `ollama pull llama3.1`
2. Get a free Tavily API key at tavily.com.
3. In n8n: **Settings → Community Nodes → Install** → `@tavily/n8n-nodes-tavily`.
4. **Workflows → Import from File →** `Transaction_Risk_MAS_Workflow.json`.
5. Credentials:
   - *Ollama Chat Model* → Ollama credential, Base URL `http://127.0.0.1:11434`, model `llama3.1:latest`.
   - *Tavily Search* → new Tavily credential with your API key.
6. Click **Open chat** and send a test case. Expect several minutes per case on a laptop. The **Logs** panel (or **Executions**) shows each agent's intermediate output and the Tavily calls.

## 7. Test Run

**Case sent in chat:**
> A savings account opened 3 weeks ago by a 20-year-old student received 26 UPI credits of ₹8,000–₹9,900 from different people today, and 95% was transferred out within an hour to two other accounts. The customer's registered email was changed yesterday.

**What each agent did (from the execution log):**

| Agent | Output |
|---|---|
| Agent 1 – Case Intake | Listed the case facts, missing information (counterparty accounts, reason for email change, usual behaviour), 4 red flags, candidate typologies (**money mule**, structuring, first-party fraud) and search keywords ("UPI money mule", "student account suspicious activity", "email change activity in banking") |
| Agent 2 – Typology Research | Called the **Tavily Search** tool 3 times with those keywords and summarised the findings |
| Agent 3 – Risk Decision | Final assessment: **risk score 85**, decision **BLOCK & REPORT** (freeze debits, escalate to the Principal Officer, consider an STR to FIU-IND), immediate actions, information to collect and a case note |

The whole pipeline ran end to end in about 10 minutes on a laptop (three sequential `llama3.1` 8B calls plus three web searches).

### Observations
- The **sequential hand-off works**: each agent received the original case and the previous agents' output, as the execution log shows.
- Tool use was confined to Agent 2, as designed.
- Limitations seen with an 8B local model: Agent 3 labelled score 85 as *Critical* (the scale puts 85 in *High*), leaned towards "phishing" because of the email change even though Agent 1 had identified a money-mule pattern, and listed sources without URLs. Possible improvements: a rule that the band must match the score, a rule to pick the typology from case facts first, only allowing URLs returned by Tavily as sources, or a larger model.

### Other cases the system is designed for
| Case | Expected outcome |
|---|---|
| *(follow-up)* What if the student says these were payments from friends for a college fest? | Agent 1 rebuilds the case from memory; the pattern still needs verification |
| A 68-year-old pensioner transferred ₹4.5 lakh by RTGS to a new beneficiary after a video call with someone claiming to be from the CBI | Digital-arrest scam, customer as victim; recall funds, 1930 helpline |
| A salaried customer paid ₹1,850 for a food order on their usual card and phone in Pune | Low; CLEAR |

## 8. Screenshots (in `screenshots/`)

| File | What it shows |
|---|---|
| `01-workflow-canvas.png` | Full workflow after the run: chat trigger → Agent 1 → Agent 2 → Agent 3, all green, with shared Ollama model, memory and Tavily |
| `02-chat-final-assessment.png` | Final assessment from Agent 3 in the chat, with the execution log of all three agents |
| `03-tavily-call.png` | Tavily Search tool call with query and results |

## 9. Why Multi-Agent

- **Separation of concerns:** each agent has a short, focused prompt, which matters more for an 8B local model than for a large hosted one.
- **Fact vs. evidence vs. judgement:** Agent 1 records only stated facts, Agent 2 adds sourced external evidence, and Agent 3 makes the call. An auditor can see where each part of the conclusion came from.
- **Tool use where it is needed:** only the research agent searches the web, which keeps the other steps fast and deterministic.
- **Human in the loop:** Agent 3 recommends; the decision bands map to who must act, and the prompt forbids tipping off the customer.

## 10. Files

| File | Description |
|---|---|
| `Transaction_Risk_MAS_Workflow.json` | Importable n8n workflow |
| `Prompts.md` | System prompts of all three agents |
| `screenshots/` | Proof of execution |
