# AIOps Monitoring and Event Processing Assessment

## Scenario
This repository simulates a lightweight AIOps workflow for a payment service operating in a production environment. The service exposes operational telemetry including request response time, CPU usage, memory usage, and structured log events. The team needs to identify abnormal traffic patterns or infrastructure stress early and route those issues through a simple event pipeline so they can be investigated and acted on.

The purpose of this assessment is to demonstrate how operational data can move through the following flow:

Operational Data → Anomaly Detection → Event Generation → Producer → Topic → Consumer → AIOps Output

## Operational Data Description
The dataset in `data/service_data.json` contains synthetic records for the `payment-service` over a ten-minute window. Each record includes:

- `timestamp`: the time the observation was recorded
- `service`: the service name emitting the telemetry
- `response_time_ms`: request latency in milliseconds
- `cpu_percent`: CPU utilization percentage
- `memory_percent`: memory utilization percentage
- `log_level`: the severity level of the service log entry (`INFO`, `WARNING`, or `ERROR`)
- `message`: the log message associated with the record

## Observations from the Metrics and Logs
Normal behaviour is represented by steady request latencies around 120-150 ms, CPU and memory values below 60%, and informational log events such as `INFO` messages indicating successful payment processing.

Unusual behaviour starts when the service begins to show latency spikes and higher resource consumption. The abnormal records are the ones with:

- `response_time_ms` above 500 ms
- `cpu_percent` above 80%
- `memory_percent` above 80%
- `log_level` values such as `ERROR` associated with timeout or connection issues

The clearest anomaly windows appear at `2026-09-20T10:05:00` and `2026-09-20T10:06:00`, where performance degradation and error logs align with a service incident.

## Anomaly Detection Findings
The `AnomalyDetector` in `src/anomaly_detector.py` reviews each record and flags a record as anomalous when any of the relevant thresholds are exceeded or when an error-level log is present. The detector returns a structured anomaly event with:

- `timestamp`
- `service`
- `type` (`ANOMALY`)
- `reasons`
- `source`

The detection result correctly identifies the abnormal records and preserves the evidence for each flagged event. The relevant signal in this data is that the earlier records are normal while the later records indicate performance degradation and runtime failures.

## Event Processing Flow
The workflow uses a lightweight in-memory pipeline:

- `EventTopic` stores the event messages in memory as a topic
- `EventProducer` publishes anomaly events to the topic
- `EventConsumer` reads the published events from the same topic
- the downstream AIOps component consumes the anomaly events and reports them as operational issues

This model is intentionally simple but mirrors the core event-streaming pattern of publish and consume.

## Final Workflow Result
Running the end-to-end pipeline produces two detected anomaly events from the ten records in the operational dataset. The final output identifies the `payment-service` degradation and the associated timeout problems at the relevant timestamps.

## Issues Identified and Corrected
During the assessment, three issues were corrected to make the workflow function properly:

1. The project package did not include an `__init__.py`, which prevented imports from `src` during test collection.
2. The producer and consumer were attached to different topics, so the consumer was unable to read the published events.
3. The anomaly detector incorrectly checked for `WARNING` logs instead of the more relevant `ERROR` severity used in the operational data.

## Limitations and Improvements
A basic threshold-based detector is useful for a small simulation, but it is limited because it relies on static thresholds and single-record checks. In a real service, an AIOps system would likely use trend analysis, seasonal baselines, and incident correlation to reduce false positives and better distinguish between transient spikes and genuine service faults.

## Reproduction Steps
1. Open the repository in GitHub Codespaces or VS Code.
2. Install dependencies from `requirements.txt` if needed.
3. Run the tests:
   `pytest -q`
4. Run the pipeline:
   `python -m src.aiops_pipeline`
5. Review the output to confirm the anomaly count and detected issues.

## Validation Summary
The repository validation confirms that the dataset is processable, anomalies are identified, events are generated, and the event pipeline passes messages through the consumer into the final AIOps result.

---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)
