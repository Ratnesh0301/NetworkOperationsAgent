# Telecom NOC Agentic Copilot (POC)

## Overview

This project is a Proof of Concept (POC) Telecom NOC Agentic Copilot built using Python, Jupyter Notebook, vLLM, and Qwen3.

The objective is to demonstrate how AI agents can assist Network Operations Center (NOC) engineers in analyzing network alarms and logs, identifying probable root causes, and generating remediation recommendations.

This implementation focuses on a single high-value telecom scenario:

**Cisco Router BGP Neighbor Down**

The solution is intentionally lightweight and can be built within a day while still showcasing an agentic workflow — including the error-handling and resilience patterns required to run structured-output agents reliably against a self-hosted LLM.

---

# Business Problem

NOC engineers often receive hundreds or thousands of alarms daily.

A typical incident may generate:

* Interface Down alarms
* BGP Neighbor Down alarms
* Route Withdrawal messages
* Link Failure notifications

Engineers must manually:

1. Review alarms
2. Analyze logs
3. Correlate events
4. Determine root cause
5. Execute remediation steps
6. Validate service restoration

This process is repetitive and time-consuming.

The Telecom NOC Copilot automates much of this workflow.

---

# Solution Architecture

```text
Alarm Input
     │
     ▼

Alarm Agent
     │
     ▼

Log Agent
     │
     ▼

Correlation Engine
     │
     ▼

RCA Agent (Qwen3, structured output)
     │
     ▼

Remediation Agent (Qwen3 + Cisco Command Library, structured output)
     │
     ▼

Summary Agent
     │
     ▼

Incident Report
```

Both LLM-backed agents (RCA Agent, Remediation Agent) follow a **try → structured output, fallback → degrade gracefully** pattern rather than letting an LLM hiccup take down the whole pipeline. See [Reliability & Error Handling](#reliability--error-handling) below.

---

# Technology Stack

| Component               | Technology                          |
| ------------------------ | ------------------------------------ |
| Language                 | Python 3.12                          |
| Development Environment  | Jupyter Notebook                     |
| LLM                      | Qwen3-4B (swappable for Qwen3-8B-Instruct with more GPU headroom) |
| Inference Server         | vLLM (`--enable-auto-tool-choice --tool-call-parser hermes`) |
| LLM Interface             | LangChain OpenAI Client (`ChatOpenAI`) |
| Structured Output Method | `with_structured_output(..., method="function_calling")` |
| Validation                | Pydantic v2                         |
| Async Support             | asyncio / `nest_asyncio`            |
| Output Format              | Structured JSON (via Pydantic models) |

---

# Project Scope

Current POC supports:

* Cisco Router alarms
* Cisco Router logs
* BGP Neighbor Down incidents
* Interface Down events
* Route Withdrawal events
* Root Cause Analysis
* Remediation Recommendations
* Graceful fallback when the LLM call fails or is truncated

Future versions may support:

* OSPF Failures
* Interface Flapping
* SD-WAN
* Switches
* Firewalls
* Linux Servers
* Multi-device Correlation

---

# Notebook Structure

The notebook is divided into four parts.

## Part 1

Foundation Layer

Contains:

* Imports
* Configuration
* vLLM Connection (`get_llm()`)
* Pydantic Models
* Alarm Agent
* Log Agent

Outputs:

```python
alarm_result

log_result
```

---

## Part 2

Root Cause Analysis

Contains:

* Correlation Engine
* RCA Models
* RCA Agent

Outputs:

```python
rca_result
```

---

## Part 3

Remediation and Summary

Contains:

* Remediation Agent
* Cisco Command Library
* Summary Agent

Outputs:

```python
remediation_result

summary_result
```

---

## Part 4

Orchestration

Contains:

* Incident Analyzer

Outputs:

```python
incident_report
```

---

# Agent Design

## 1. Alarm Agent

### Purpose

Normalize incoming alarm information.

### Input

```json
{
  "device": "RTR-01",
  "event": "BGP_NEIGHBOR_DOWN",
  "severity": "CRITICAL"
}
```

### Output

```json
{
  "incident_type": "Network Routing Failure",
  "severity": "CRITICAL",
  "event": "BGP_NEIGHBOR_DOWN"
}
```

### Responsibilities

* Parse alarm
* Normalize event types
* Categorize incident

Deterministic, rule-based — no LLM call.

---

## 2. Log Agent

### Purpose

Analyze router logs and identify patterns.

### Input

```text
Interface GigabitEthernet0/0 down
BGP neighbor down
Route withdrawal initiated
```

### Output

```json
{
  "interface_down": true,
  "bgp_neighbor_down": true,
  "route_withdrawal": true
}
```

### Responsibilities

* Pattern detection
* Event extraction
* Evidence generation

### Detection Rules

| Pattern          | Meaning               |
| ---------------- | --------------------- |
| Interface Down   | Physical link failure |
| BGP Down         | Routing failure       |
| Route Withdrawal | Traffic impact        |

Deterministic, regex-based — no LLM call.

---

## 3. Correlation Engine

### Purpose

Convert raw findings into contextual information before sending to the LLM.

### Example

Input:

```text
Interface Down
BGP Neighbor Down
Route Withdrawal
```

Output:

```text
Scenario:
BGP Failure caused by Interface Failure

Probable Cause:
WAN Interface Outage
```

### Why Use Correlation?

Without correlation:

```text
Raw Logs → LLM
```

With correlation:

```text
Logs → Correlation → LLM
```

This reduces hallucinations and improves RCA quality. It also doubles as the **fallback root cause** if the RCA Agent's LLM call fails (see below).

---

## 4. RCA Agent

### Purpose

Determine the most probable root cause.

### Model

Qwen3 running through vLLM, called via `with_structured_output(RCAResult, method="function_calling")`.

### Input

* Alarm Analysis
* Log Analysis
* Correlation Results

### Output

```json
{
  "root_cause": "WAN Interface Failure",
  "impacted_component": "GigabitEthernet0/0",
  "confidence": 0.94,
  "reasoning": "Interface failure preceded BGP loss."
}
```

### Responsibilities

* Root Cause Analysis
* Confidence Scoring
* Technical Explanation

### Confidence Scoring

`confidence` is typed as a `float` with `Field(ge=0.0, le=1.0)` — the schema itself constrains the model to return a number between 0 and 1, rather than a free-text label like `"High"`/`"Low"`. This prevents a downstream type mismatch when the value later flows into `IncidentSummary` and `IncidentReport`, both of which expect a numeric confidence.

### Failure Handling

If the structured-output call fails for any reason (truncated response, schema violation, etc.), the agent falls back to the `CorrelationEngine`'s rule-based guess, sets `confidence=0.0`, and continues the pipeline rather than raising.

---

## 5. Remediation Agent

### Purpose

Generate corrective actions.

### Input

```json
{
  "root_cause": "WAN Interface Failure"
}
```

### Output

```json
{
  "actions": [
    "Verify WAN connectivity",
    "Inspect optical signal levels"
  ],
  "commands": [
    "show interfaces GigabitEthernet0/0",
    "show ip bgp summary"
  ],
  "validation_steps": [
    "Verify interface state is UP",
    "Verify BGP neighbor established"
  ]
}
```

### Responsibilities

* Troubleshooting guidance (`actions`, LLM-generated)
* CLI commands and validation checks (`commands`, `validation_steps` — looked up deterministically from the Cisco Command Library, **not** generated by the LLM)

### Design Notes

* Uses `with_structured_output(RemediationResult, method="function_calling")`, matching the server's `--tool-call-parser hermes` configuration.
* `max_tokens` is capped at 1024 for this agent (512 for its fallback call) since the expected output is a handful of short actions. If generation ever runs away again, it now fails fast and cheap instead of burning a full 8192-token budget.
* If structured output fails, the agent retries with a plain-text prompt and parses a numbered list out of the response. If that also fails, it returns a safe `"Manual review required"` action so the incident report is still produced.

---

## 6. Cisco Command Library

### Purpose

Provide deterministic commands.

Instead of allowing the LLM to invent commands, known Cisco commands are stored locally and looked up by `root_cause`.

### Example

```python
show interfaces GigabitEthernet0/0

show logging

show ip bgp summary
```

### Benefits

* Predictable output
* Safer recommendations
* Reduced hallucination risk

---

## 7. Summary Agent

### Purpose

Generate a concise incident report.

### Input

* RCA Results
* Remediation Results

### Output

Executive summary suitable for:

* NOC dashboards
* Shift handovers
* Incident tickets

Deterministic, template-based — no LLM call.

---

## 8. Incident Analyzer

### Purpose

Orchestrate the entire workflow.

### Execution Sequence

```text
Alarm Agent
     ↓
Log Agent
     ↓
Correlation Engine
     ↓
RCA Agent
     ↓
Remediation Agent
     ↓
Summary Agent
```

### Output

```json
{
  "incident_type": "Network Routing Failure",
  "root_cause": "WAN Interface Failure",
  "confidence": 0.94,
  "actions": [
    "Verify WAN connectivity"
  ]
}
```

---

# Reliability & Error Handling

This POC originally hit a `LengthFinishReasonError` from the OpenAI SDK on the Remediation Agent:

```text
LengthFinishReasonError: Could not parse response content as the length
limit was reached - CompletionUsage(completion_tokens=8192, prompt_tokens=95, ...)
```

### Root Cause

`ChatOpenAI.with_structured_output(schema)` defaults to `method="json_schema"`, which uses OpenAI's native Structured Outputs feature (schema-constrained decoding via `client.chat.completions.parse()`). The vLLM server, however, is started with `--enable-auto-tool-choice --tool-call-parser hermes`, which configures it for **tool/function calling**, not schema-constrained decoding. The mismatch caused the small local model to never emit a stop token on certain prompts, generating tokens until it hit the `max_tokens` ceiling (8192) with no valid output to parse.

### Fix

* Both LLM-backed agents (RCA Agent, Remediation Agent) explicitly use `method="function_calling"`, which routes structured output through the hermes tool-call parser the server actually understands.
* The Remediation Agent's prompt was simplified — conflicting instructions like "Output STRICT JSON only" plus repeated "do not explain" lines were removed, since they fight against the tool-calling mechanism.
* Both agents wrap their LLM call in `try/except` and degrade gracefully on failure instead of crashing the whole `IncidentAnalyzer.analyze()` run:
  * **RCA Agent** → falls back to the `CorrelationEngine`'s rule-based guess with `confidence=0.0`.
  * **Remediation Agent** → falls back to a plain-text numbered-list parse, then to a static `"Manual review required"` action if that also fails.
* `max_tokens` is now tuned per agent (1024 / 512 for Remediation vs. the 8192 default) so a similar runaway fails fast and cheaply rather than burning the full budget.

### Secondary Bug Fixed

`RCAResult.confidence` and `IncidentSummary.confidence` were originally typed `str`, while `IncidentReport.confidence` was typed `float`. Since `IncidentAnalyzer.analyze()` never normalized the value, a model response like `"High"` instead of `0.85` would raise a Pydantic `float_parsing` `ValidationError` deep in the Summary step. `confidence` is now `float` end-to-end, constrained at the schema level (`ge=0.0, le=1.0`), with a `normalize_confidence()` helper kept as a defensive no-op safety net.

---

# vLLM Configuration

The notebook assumes vLLM is already running locally, started with:

```bash
VLLM_USE_TRITON_FLASH_ATTN=0 vllm serve Qwen/Qwen3-4B \
  --served-model-name Qwen3-4B \
  --api_key abc-123 \
  --port 8000 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes \
  --trust-remote-code \
  --max_model_len 24272
```

Endpoint:

```text
http://0.0.0.0:8000/v1
```

Model:

```text
Qwen3-4B
```

Connection:

```python
ChatOpenAI(
    model="Qwen3-4B",
    base_url="http://0.0.0.0:8000/v1",
    api_key="abc-123",
    temperature=0,
    max_tokens=8192,
    extra_body={
        "chat_template_kwargs": {
            "enable_thinking": False
        }
    }
)
```

`--enable-auto-tool-choice --tool-call-parser hermes` is what makes `method="function_calling"` (see [Reliability & Error Handling](#reliability--error-handling)) the correct structured-output strategy for this server.

---

# Sample Test Scenario

## Alarm

```json
{
  "device": "RTR-01",
  "event": "BGP_NEIGHBOR_DOWN",
  "severity": "CRITICAL"
}
```

## Logs

```text
Interface GigabitEthernet0/0 down

BGP-5-ADJCHANGE neighbor 192.168.1.1 Down

Route withdrawal initiated
```

---

# Expected Analysis

## Correlation

```text
BGP Failure caused by Interface Failure
```

## RCA

```text
WAN Interface Failure
```

## Remediation

```text
Verify interface state
Check optical power
Validate BGP session
```

---

# Running the Notebook

Execute cells in order:

```text
Part 1
↓
Part 2
↓
Part 3
↓
Part 4
```

Then run:

```python
incident_report = await analyzer.analyze(
    alarm,
    logs
)
```

If you've just pulled the fixed version of the notebook, do a **Restart Kernel & Run All** rather than re-running individual cells out of order — all outputs have been cleared so cell numbering and state stay consistent.

---

# Current Limitations

This is a POC and intentionally simplified.

Not included:

* LangGraph
* ChromaDB
* RAG
* Multi-device correlation
* Historical memory
* Human approval workflows
* Streaming responses
* OpenTelemetry tracing

Included (lightweight resilience, not full production-grade reliability):

* Try/except fallback on both LLM-backed agents
* Per-agent `max_tokens` budgeting
* Schema-level confidence validation

---

# Future Enhancements

## Phase 2

Add:

* OSPF Failures
* Interface Flaps
* Multiple Routers
* Switch Logs

## Phase 3

Add:

* LangGraph
* ChromaDB
* Incident Memory
* Historical Search
* Tool Calling

## Phase 4

Add:

* Streamlit Dashboard
* FastAPI Backend
* Authentication
* Incident Ticket Integration

---

# Conclusion

This project demonstrates a practical Telecom NOC Agentic Copilot capable of:

* Understanding network alarms
* Parsing router logs
* Correlating events
* Performing AI-assisted root cause analysis
* Recommending remediation actions
* Generating incident summaries
* Degrading gracefully when the local LLM serving stack misbehaves

while remaining simple enough to implement, understand, and demonstrate within a single day.
