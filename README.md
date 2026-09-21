# AIOps Operational Data Analysis and Workflow Validation

## 1. AIOps scenario

This repository simulates a lightweight AIOps workflow for a payment service. The service records operational metrics and log entries over a short time window, then applies anomaly detection to identify abnormal behaviour. The workflow models the flow of a detected issue through an in-memory event stream so that a service degradation can be represented as an event, published to a topic, consumed, and passed to downstream processing.

The goal is to distinguish healthy runtime behaviour from a service incident by combining:
- numeric telemetry (`response_time_ms`, `cpu_percent`, `memory_percent`)
- log context (`log_level`, `message`)
- timestamped operational records

## 2. Description of the operational data

The repository contains a synthetic dataset in `data/service_data.json`.

Each record contains:
- `timestamp`: ISO timestamp for the event time
- `service`: application/service name
- `response_time_ms`: latency in milliseconds
- `cpu_percent`: CPU utilization percentage
- `memory_percent`: memory utilization percentage
- `log_level`: log severity (`INFO`, `ERROR`)
- `message`: explanatory log text

The dataset covers 10 observations from `2026-09-20T10:00:00` through `2026-09-20T10:09:00` for the `payment-service`.

## 3. Observations from the logs and metrics

### Metric fields
The metric fields are:
- `response_time_ms`
- `cpu_percent`
- `memory_percent`

They show the runtime health of the service over time.

### Log fields
The log-related fields are:
- `timestamp`
- `service`
- `log_level`
- `message`

These provide the contextual explanation of operational state.

### Timestamps
The timestamps are ISO 8601 values recorded once per minute. They allow the records to be read in chronological order and correlate spikes in metrics with the corresponding log entries.

### Normal behaviour
These observations are considered normal:
- `10:00`, `10:01`, `10:02`, `10:03`, `10:04`
- `10:07`, `10:08`, `10:09`

Typical values:
- `response_time_ms` around `120-145 ms`
- `cpu_percent` around `42-50%`
- `memory_percent` around `51-57%`
- `log_level` is `INFO`
- log message: `Payment request processed successfully`

This pattern indicates steady service performance.

### Unusual behaviour
The unusual observations are:
- `2026-09-20T10:05:00`
- `2026-09-20T10:06:00`

These records show:
- extremely high latency: `610` ms and `640` ms
- CPU values of `75%` and `94%`
- memory values of `70%` and `91%`
- `ERROR` log severity
- messages such as `Payment service timeout` and `Database connection timeout`

These readings indicate service degradation and an operational incident.

## 4. Anomaly-detection findings

The repository’s anomaly detector flags records whose metrics exceed the configured thresholds and whose log level indicates concern.

The two anomalous observations identified are:

1. `2026-09-20T10:05:00`
   - `response_time_ms = 610`
   - `cpu_percent = 75`
   - `memory_percent = 70`
   - `log_level = ERROR`
   - message: `Payment service timeout`
   - reasons: `High response time`, `Error log detected`

2. `2026-09-20T10:06:00`
   - `response_time_ms = 640`
   - `cpu_percent = 94`
   - `memory_percent = 91`
   - `log_level = ERROR`
   - message: `Database connection timeout`
   - reasons: `High response time`, `High CPU utilization`, `High memory utilization`, `Error log detected`

These events align with the abnormal values and are the expected anomaly window.

## 5. Event-processing flow

The event-processing flow in this project is:

Operational Data -> Anomaly Detection -> Event -> Producer -> Topic -> Consumer -> AIOps

### Component roles
- `Producer`: takes a generated anomaly event and publishes it to the topic.
- `Topic`: stores the event in memory.
- `Consumer`: reads events from the topic.
- `Event/message`: the structured payload that carries timestamp, service, reasons, and source telemetry/log context.

## 6. Final workflow execution result

The corrected workflow was executed successfully with:

```bash
cd /workspaces/github-skills-challenge
PYTHONPATH=src python - <<'PY'
import json
from anomaly_detector import AnomalyDetector
from event_producer import EventProducer
from event_consumer import EventConsumer
from event_topic import EventTopic

with open('data/service_data.json', 'r', encoding='utf-8') as f:
    data = json.load(f)

anomaly_topic = EventTopic('anomaly-events')
producer = EventProducer(anomaly_topic)
consumer = EventConsumer(anomaly_topic)
detector = AnomalyDetector()
processed_records = []
produced_events = []

for record in data:
    processed_records.append(record)
    event = detector.detect(record)
    if event is not None:
        produced_events.append(event)
        producer.publish(event)

consumed_events = consumer.consume()

print('Operational Data Processed:', len(processed_records))
print('Anomalies Detected:', len(produced_events))
print('Anomaly Events Generated:', len(produced_events))
print('Events Published:', len(anomaly_topic.get_messages()))
print('Events Consumed:', len(consumed_events))
print('Processed Successfully:', len(consumed_events) == len(produced_events))

for event in consumed_events:
    print(f"Service: {event['service']}")
    print(f"Timestamp: {event['timestamp']}")
    print(f"Type: {event['type']}")
    print(f"Reasons: {', '.join(event['reasons'])}")
    print(f"Source: {event['source']['log_level']} | {event['source']['message']}")
PY
```

Observed result:

```text
Operational Data Processed: 10
Anomalies Detected: 2
Anomaly Events Generated: 2
Events Published: 2
Events Consumed: 2
Processed Successfully: True
```

Final AIOps output:

```text
Service: payment-service
Timestamp: 2026-09-20T10:05:00
Type: ANOMALY
Reasons: High response time, Error log detected
Source: ERROR | Payment service timeout

Service: payment-service
Timestamp: 2026-09-20T10:06:00
Type: ANOMALY
Reasons: High response time, High CPU utilization, High memory utilization, Error log detected
Source: ERROR | Database connection timeout
```

This confirms the complete event flow works and that the final output represents the detected operational issue.

## 7. Issues identified and corrected

Two problems affected the workflow:

### Issue 1: mismatched topic names
- Affected component: pipeline/event stream
- Cause: the producer published to `service-events` while the consumer read from `anomaly-events`
- Correction: both components now use the same topic name: `anomaly-events`
- Result: events are successfully published and consumed

### Issue 2: log severity mismatch
- Affected component: anomaly detector
- Cause: the detector only considered `WARNING` logs, while the dataset uses `ERROR` log entries
- Correction: treat both `WARNING` and `ERROR` as concerning log severities
- Result: the real timeout events are correctly classified as anomalies

## 8. Limitation or possible improvement

The current AIOps approach uses fixed thresholds for latency, CPU, and memory. This is simple and useful for a synthetic demo, but it has a limitation: it may not adapt well to different services or changing workload patterns. A stronger approach would be to make thresholds configurable per service, add a moving-window baseline, and evaluate log severity alongside a larger set of operational signals.

## 9. Steps to reproduce the demonstration

To reproduce this work in a fresh checkout:

1. Open a terminal in the repository root.
2. Create or activate the Python environment.
3. Install project dependencies:

```bash
pip install -r requirements.txt
```

4. Run the AIOps workflow directly:

```bash
PYTHONPATH=src python src/aiops_pipeline.py
```

5. Verify the output shows the detected anomalies and the event flow result.
6. Run the test suite:

```bash
PYTHONPATH=. pytest -q
```

If the repository is run with the project root on the Python path, the import structure works correctly. In this environment, the `PYTHONPATH` setting is required for reliable module resolution.

## Summary

This project demonstrates a small but realistic AIOps workflow: operational telemetry and logs are analyzed, anomalies are detected, events are published and consumed, and the final result clearly identifies the service issue. The corrected workflow now processes all data, detects the two timeout incidents, publishes them as events, consumes them successfully, and produces a final downstream anomaly output that reflects the actual operational problem.

---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)
