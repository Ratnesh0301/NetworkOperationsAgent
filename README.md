# Telecom NOC Agentic Copilot (POC)

## Overview

This project is a Proof of Concept (POC) Telecom NOC Agentic Copilot built using Python, Jupyter Notebook, vLLM, and Qwen3.

The objective is to demonstrate how AI agents can assist Network Operations Center (NOC) engineers in analyzing network alarms and logs, identifying probable root causes, and generating remediation recommendations.

This implementation focuses on a single high-value telecom scenario:

**Cisco Router BGP Neighbor Down**

The solution is intentionally lightweight and can be built within a day while still showcasing an agentic workflow.

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

RCA Agent (Qwen3)
     │
     ▼

Remediation Agent
     │
     ▼

Summary Agent
     │
     ▼

Incident Report
```

---

# Technology Stack

| Component               | Technology              |
| ----------------------- | ----------------------- |
| Language                | Python 3.12             |
| Development Environment | Jupyter Notebook        |
| LLM                     | Qwen3-8B-Instruct       |
| Inference Server        | vLLM                    |
| LLM Interface           | LangChain OpenAI Client |
| Validation              | Pydantic                |
| Async Support           | asyncio                 |
| Output Format           | Structured JSON         |

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
* vLLM Connection
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

This reduces hallucinations and improves RCA quality.

---

## 4. RCA Agent

### Purpose

Determine the most probable root cause.

### Model

Qwen3 running through vLLM.

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
  ]
}
```

### Responsibilities

* Troubleshooting guidance
* Recovery recommendations

---

## 6. Cisco Command Library

### Purpose

Provide deterministic commands.

Instead of allowing the LLM to invent commands, known Cisco commands are stored locally.

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

# vLLM Configuration

The notebook assumes vLLM is already running locally.

Endpoint:

```text
http://localhost:8000/v1
```

Model:

```text
Qwen/Qwen3-8B-Instruct
```

Connection:

```python
ChatOpenAI(
    model="Qwen/Qwen3-8B-Instruct",
    base_url="http://localhost:8000/v1",
    api_key="EMPTY"
)
```

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

while remaining simple enough to implement, understand, and demonstrate within a single day.
