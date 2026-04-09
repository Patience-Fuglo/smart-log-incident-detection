# 🖥️ Smart Log Processing & Incident Detection System

A pure-Python log-processing pipeline that ingests raw, messy server log data, validates and cleans each entry, classifies incident severity, and produces a structured incident report — built entirely without external libraries.

---

## 🧩 Project Overview

Production server environments emit thousands of log entries per minute, and those logs are rarely clean. Server names arrive with stray whitespace, log levels come in mixed case, messages can be `None`, and usage values are sometimes non-numeric strings like `"high"`. A pipeline that crashes on any of these is useless in production.

This project builds a **defensive, modular log-processing engine** that handles all of these failure modes gracefully — skipping invalid entries, logging every attempt, and surfacing a clean structured output for only the records that pass validation.

---

## 🎯 What It Does

Given raw log entries in the format:

```python
[server_name, log_level, message, usage_percentage]
```

The pipeline:

1. **Cleans** server names (strips whitespace, normalises to uppercase)
2. **Normalises** log levels (uppercase + allowlist check — only `ERROR`, `WARNING`, `INFO` are valid)
3. **Validates** log messages (rejects `None` and non-string values)
4. **Validates** usage percentages (must be numeric and within 0–100)
5. **Classifies** severity using a rule matrix (see below)
6. **Detects** whether the entry constitutes an incident
7. **Skips** invalid entries without crashing — always logs `"Log Processed"` via `finally`
8. **Aggregates** a severity-broken-down incident summary across the full log batch

---

## 🗂️ Project Structure

```
smart-log-incident-detection/
│
├── PROJECT_Smart_Log_Processing_Incident_Detection_System.ipynb
└── README.md
```

---

## 🔧 Core Functions

| Function | Responsibility |
|---|---|
| `clean_server_name(name)` | Strips whitespace, converts to uppercase |
| `normalize_log_level(level)` | Uppercases and validates against `['ERROR', 'WARNING', 'INFO']`; raises `ValueError` for anything else (e.g. `DEBUG`) |
| `validate_message(message)` | Raises `TypeError` if message is `None` or not a string |
| `validate_usage(usage)` | Raises `TypeError` if not numeric; raises `ValueError` if outside 0–100 |
| `classify_severity(level, usage)` | Applies severity rule matrix (see below) |
| `detect_incident(severity)` | Returns `True` if severity is anything other than `LOW` |
| `process_single_log(raw_log)` | Orchestrates all steps for one entry; returns `None` on any failure |
| `process_all_logs(raw_logs)` | Iterates the full batch; collects only valid processed log dicts |
| `incident_summary(logs)` | Returns total count, incident count, and severity-wise breakdown |

---

## 📊 Severity Classification Matrix

| Log Level | Usage Threshold | Severity |
|---|---|---|
| `ERROR` | ≥ 90% | `CRITICAL` |
| `ERROR` | ≥ 75% | `HIGH` |
| `ERROR` | < 75% | `LOW` |
| `WARNING` | ≥ 80% | `MEDIUM` |
| `WARNING` | < 80% | `LOW` |
| `INFO` | any | `LOW` |

An entry is flagged as an **incident** (`incident: True`) whenever severity is `CRITICAL`, `HIGH`, or `MEDIUM`.

---

## ▶️ Sample Run

**Input:**
```python
raw_logs = [
    [" server-1 ", "ERROR",   "Disk space low",    92],   # valid — CRITICAL
    ["SERVER-2",   "warning", "High memory usage",  78],   # valid — LOW (WARNING < 80)
    ["server-3",   "INFO",    "Backup completed",   40],   # valid — LOW
    ["server-4",   "ERROR",   None,                 95],   # rejected — None message
    ["server-5",   "ERROR",   "CPU overheating",  "high"], # rejected — non-numeric usage
    [" server-6",  "WARNING", "Network latency",    81],   # valid — MEDIUM
    ["server-7",   "DEBUG",   "Heartbeat",          10]    # rejected — invalid log level
]
```

**Console output during processing:**
```
Log Processed
Log Processed
Log Processed
Error processing log '['server-4', 'ERROR', None, 95]': Message cannot be None.
Log Processed
Error processing log '['server-5', 'ERROR', 'CPU overheating', 'high']': Usage must be a number.
Log Processed
Log Processed
Error processing log '['server-7', 'DEBUG', 'Heartbeat', 10]': Invalid log level: DEBUG. Allowed levels are ERROR, WARNING, INFO.
Log Processed
```

**Processed log output (4 of 7 entries pass):**
```python
[
  {'server': 'SERVER-1', 'level': 'ERROR',   'message': 'Disk space low',    'usage': 92, 'severity': 'CRITICAL', 'incident': True},
  {'server': 'SERVER-2', 'level': 'WARNING', 'message': 'High memory usage', 'usage': 78, 'severity': 'LOW',      'incident': False},
  {'server': 'SERVER-3', 'level': 'INFO',    'message': 'Backup completed',  'usage': 40, 'severity': 'LOW',      'incident': False},
  {'server': 'SERVER-6', 'level': 'WARNING', 'message': 'Network latency',   'usage': 81, 'severity': 'MEDIUM',   'incident': True}
]
```

**Incident summary:**
```python
{
    'total_logs': 4,
    'incidents': 2,
    'severity_count': {
        'CRITICAL': 1,
        'HIGH':     0,
        'MEDIUM':   1,
        'LOW':      2
    }
}
```

---

## 🛡️ Error Handling Strategy

Three distinct rejection scenarios are handled, each with a different exception type:

| Failure Mode | Example Input | Exception Raised |
|---|---|---|
| `None` message | `["server-4", "ERROR", None, 95]` | `TypeError` |
| Non-numeric usage | `["server-5", "ERROR", "msg", "high"]` | `TypeError` |
| Invalid log level | `["server-7", "DEBUG", "msg", 10]` | `ValueError` |

All exceptions are caught by a single broad `except Exception as e` in `process_single_log`, which prints the error with context and returns `None`. The `finally` block fires unconditionally, printing `"Log Processed"` for every entry regardless of outcome — providing a complete audit trail of all attempted records.

---

## 💡 Key Design Decisions

**Allowlist validation over silently ignoring** — `normalize_log_level` explicitly raises a `ValueError` for `DEBUG` rather than silently dropping it. This makes rejection reasons visible and debuggable, which matters in real ops pipelines.

**Type checking before range checking** — `validate_usage` checks `isinstance(usage, (int, float))` before doing `usage < 0 or usage > 100`. This avoids a `TypeError` from comparing a string to an integer, and produces a cleaner, more informative error message.

**`finally` as an audit mechanism** — every log entry, valid or not, triggers `"Log Processed"`. In a production context this would be a structured log line, giving ops teams a complete record of what the pipeline attempted vs. what it accepted.

**`None`-safe message validation** — `validate_message` checks for `None` first, before the `isinstance` check. Reversing that order would cause `isinstance(None, str)` to silently return `False`, losing the specificity of the `TypeError`.

**`incident_summary` uses dict-based aggregation** — rather than multiple pass-throughs, the function builds the full severity breakdown in one loop using `sum()` with generator expressions, keeping it readable and O(n).

---

## 🧠 Concepts Demonstrated

- Defensive programming and input validation
- Exception handling with `try / except / finally`
- Allowlist-based validation pattern
- Single-responsibility function design
- Type-aware validation (`isinstance` checks)
- Structured dictionary output
- Aggregation and reporting from processed data
- Real-world failure modes: `None` values, wrong types, out-of-range inputs, invalid enums

---

## 🚀 How to Run

No dependencies required — pure Python 3.

```bash
# Clone the repo
git clone https://github.com/Patience-Fuglo/smart-log-incident-detection.git
cd smart-log-incident-detection

# Open the notebook in Jupyter or Google Colab and run all cells
```

Or open directly in [Google Colab](https://colab.research.google.com/).

---

## 📌 Context

Built as a Week 1 project during an intensive Python programme. The scenario — server log ingestion with dirty real-world data — was chosen deliberately to mirror the kind of data quality problems that appear in SRE, data engineering, and backend systems work. The focus was on code correctness, defensive design, and producing genuinely useful structured output rather than just making the happy path work.
