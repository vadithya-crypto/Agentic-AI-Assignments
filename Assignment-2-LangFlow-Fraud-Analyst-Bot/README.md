# Assignment 2 – FraudLens: Fraud Risk Analyst Smart Bot (LangFlow + Ollama)

## 1. Project Overview

**FraudLens** is a smart chatbot built in **LangFlow** that acts as a bank's Fraud Risk Analyst. A user describes a transaction or customer behaviour in plain language, and the bot returns a structured fraud assessment: red flags, likely fraud typology, a risk score, recommended actions, and what information is still needed. It remembers the conversation, so an analyst can ask follow-up questions about the same case, and it can **search the web** when asked about a recent scam or advisory.

The model runs **locally through Ollama**, so no transaction details are sent to a cloud LLM, which matters for a banking use case.

## 2. Objectives

- Build a working LLM application with LangFlow's visual components.
- Use role-based prompt engineering to make a general model behave like a domain specialist.
- Produce a consistent, structured output (7 fixed sections) suitable for a fraud operations team.
- Use conversational memory so follow-up questions refer back to the same case.
- Give the bot tools (Web Search, Calculator, Current Date) and let the agent decide when to use them.
- Add guardrails: no invented facts, no accusations, refusal of fraud-enabling requests.

## 3. Tools and Model

| Component | Purpose |
|---|---|
| LangFlow | Visual low-code builder for the flow |
| Ollama | Runs the LLM locally |
| **LLM: `llama3.2:latest`** (Llama 3.2, 3B) | Model provider selected inside the Agent component |
| Chat Input / Chat Output | User interface in the Playground |
| **Agent** | Holds the FraudLens instructions, keeps chat memory, and decides which tool to call |
| Web Search (tool) | Looks up recent scams, RBI/NPCI advisories and fraud news |
| Calculator, Current Date (built-in Agent tools) | Arithmetic on amounts; today's date for time-based reasoning |

## 4. Workflow Architecture

```text
                                   Web Search (tool)
                                          │
                                          ▼
User ──► Chat Input ──────────────►   AGENT   ──────────────► Chat Output ──► User
                                  (FraudLens instructions,
                                   Ollama llama3.2,
                                   built-in chat memory,
                                   Calculator + Current Date tools)
```

The Agent receives the user's message together with the session's chat history, follows the FraudLens instructions, and calls Web Search only when the question needs current information.

## 5. Build Steps

1. `ollama pull llama3.2` and make sure Ollama is running (`http://localhost:11434`).
2. LangFlow → **New Flow → Blank Flow**, name it `FraudLens - Fraud Risk Analyst Bot`.
3. Add **Chat Input**, **Agent** and **Chat Output**.
4. In the Agent: Model Provider **Ollama**, Model **llama3.2:latest**; paste the instructions from `Prompts.md` into **Agent Instructions**; keep *Current Date* and *Calculator* tools on.
5. Add **Web Search**, switch on **Tool Mode**, and connect it to the Agent's **Tools** input.
6. Connect Chat Input → Agent → Chat Output.
7. Open **Playground** and run the test conversation.
8. Export: **Share → Export**, saved as `FraudLens_Flow.json` (without API keys).

## 6. Prompt Design

The full prompt is in `Prompts.md`. It has three modes:

- **Case analysis:** a fixed 7-section output (Case Summary, Red Flags, Typology, Risk Rating with score bands, Recommended Actions, Information Needed, Confidence).
- **Concept explanation:** under 200 words with an example and two prevention controls.
- **Follow-up:** uses the conversation history to resolve "this case", "the customer", and similar references.

A **Tool Use** section tells the agent to search the web only for recent scams, advisories or trends, and to mention the source. Guardrails keep the model to the facts provided, separate *suspicious* from *confirmed*, avoid naming real people, and refuse requests that would help someone commit or hide fraud.

## 7. Test Conversation

| # | Question | What it shows |
|---|---|---|
| 1 | Card fraud case: 6 purchases of ₹19,500 in 40 minutes at 2:30 AM from a new device, mobile number changed the day before | Full 7-section assessment with risk rating |
| 2 | *Follow-up:* "What if the customer confirms they changed the phone number themselves because they bought a new SIM?" | Memory: the bot revises the same case and explains the change |
| 3 | A question about a recent scam trend in India | Tool use: the agent calls Web Search and summarises the results |

## 8. Screenshots (in `screenshots/`)

| File | What it shows |
|---|---|
| `01-flow-canvas.png` | LangFlow canvas: Chat Input → Agent (with Web Search tool) → Chat Output |
| `02-case-analysis.png` | Playground: 7-section analysis for question 1 |
| `03-follow-up-memory.png` | Playground: follow-up answered using memory |
| `04-web-search.png` | Playground: recent-scam question answered with the Web Search tool |
| `05-evaluate-expression.png` | Agent calling its built-in Calculator tool (`evaluate_expression`) |

## 9. Outcome

FraudLens shows how a small, locally run model can be turned into a consistent domain assistant with prompt engineering and an agent that has tools. The fixed output structure makes responses comparable across cases, memory supports an analyst's investigation, web search adds current context, and the guardrails keep the bot useful for detection while refusing misuse.

## 10. Files

| File | Description |
|---|---|
| `FraudLens_Flow.json` | Exported LangFlow flow (import via LangFlow → Upload / Import) |
| `Prompts.md` | Full agent instructions, techniques used, and test conversation |
| `screenshots/` | Proof of execution |
