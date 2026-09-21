# GitHub Challenge

<img src="https://octodex.github.com/images/Professortocat_v2.png" align="right" height="200px" />

Hey there!

Your challenge is ready.
Follow the instructions provided for this challenge and complete the required tasks in this repository.

Make sure your work is committed and pushed to your repository before submission.

Good luck!

## Task-1 

This assessment focuses on a payment-processing service, represented in the repository as the service named payment-service. The operational data for this service includes timestamps, request response times, CPU and memory usage, and log severity information gathered from normal and degraded runtime behavior. The core operational problem is that the service can experience slowdowns and failures during peak traffic or resource saturation, which shows up as high latency, elevated resource consumption, and error-level logs that can affect payment processing reliability.

The purpose of AIOps in this assessment is to turn raw telemetry into actionable signal. Rather than requiring an operator to manually scan metrics and logs, the workflow detects suspicious patterns, publishes anomalous events to an in-memory event stream, and consumes those events for downstream reporting. In other words, AIOps here is used to detect operational issues early, correlate relevant signals, and surface likely service incidents for investigation and response.

## Task-2: Logs and Metrics Analysis

The operational dataset in `data/service_data.json` is a compact sequence of telemetry records for the `payment-service` collected at one-minute intervals. The fields are a mix of numeric service metrics and textual application logs.

### 1. Metrics fields

The metric-like fields are:

- `response_time_ms`: request latency measured in milliseconds.
- `cpu_percent`: percentage of CPU used by the service.
- `memory_percent`: percentage of memory usage.
- `service`: identifies which service emitted the observation, but it is not a metric itself.

These values are numeric and allow the service to be monitored over time for performance and capacity trends.

### 2. Log information fields

The log-related fields are:

- `log_level`: severity such as `INFO` or `ERROR`.
- `message`: human-readable log message describing the event or failure.

The logs provide context for the numeric measurements. For example, successful records include `INFO` entries such as "Payment request processed successfully", while the degraded samples include `ERROR` messages such as "Payment service timeout" and "Database connection timeout".

### 3. Use of timestamps

The `timestamp` value is recorded in ISO 8601 format (`YYYY-MM-DDTHH:MM:SS`) and is used to order the telemetry chronologically. The data shows a sequence from `2026-09-20T10:00:00` through `2026-09-20T10:09:00`, with observations sampled once per minute. This makes it possible to see when the service is healthy, when it degrades, and when it recovers.

### 4. Normal behaviour

The observations that appear normal are the first, second, third, fourth, seventh, eighth, ninth, and tenth records, depending on the sample progression after the incident. In these records:

- `response_time_ms` stays near 120-150 ms.
- `cpu_percent` remains roughly 42-50%.
- `memory_percent` stays roughly 51-57%.
- `log_level` is `INFO`.
- The message states that payment requests were processed successfully.

This pattern indicates steady operation with low latency and moderate resource usage, which is consistent with normal service behavior.

### 5. Unusual behaviour

The unusual observations are the records around `2026-09-20T10:05:00` and `2026-09-20T10:06:00`:

- `response_time_ms`: 610 ms and 640 ms, far above the normal 120-150 ms range.
- `cpu_percent`: 75% and 94%, which is much higher than the usual 42-50% range.
- `memory_percent`: 70% and 91%, indicating pressure on available memory.
- `log_level`: `ERROR` instead of `INFO`.
- `message`: indicates service and database timeout conditions.

Together, these signals suggest an incident or resource saturation event where the service is struggling to complete requests and is generating explicit failure-level logs. The later records return toward normal values, which suggests recovery after the incident.

In summary, the repository’s telemetry shows a short-lived degradation event superimposed on a mostly healthy service baseline. That pattern is exactly the kind of signal the AIOps workflow in this assessment is designed to detect and escalate.



## Task-3: Validation of Anomaly Detection and Event Streaming

The repository’s provided detector was applied to the operational data in `data/service_data.json` to validate that the AIOps workflow can identify abnormal service behavior and produce a readable detection report.

### Validation results

The detector processed all 10 records in the dataset and flagged 2 observations as anomalies. The flagged records are:

- `2026-09-20T10:05:00`
  - `response_time_ms`: 610
  - `cpu_percent`: 75
  - `memory_percent`: 70
  - `log_level`: `ERROR`
  - `message`: "Payment service timeout"
  - Reasons returned: `High response time`

- `2026-09-20T10:06:00`
  - `response_time_ms`: 640
  - `cpu_percent`: 94
  - `memory_percent`: 91
  - `log_level`: `ERROR`
  - `message`: "Database connection timeout"
  - Reasons returned: `High response time`, `High CPU utilization`, `High memory utilization`

These anomalies clearly match the operational pattern described in the data: a short-lived service degradation with elevated latency and resource pressure, accompanied by failure-level logs.

### Difference between normal and anomalous observations

The remaining observations are normal. They have response times near 120–150 ms, CPU around 42–50%, memory around 51–57%, and `INFO` log entries indicating successful payment processing. The detector does not flag those records, which is consistent with the expected baseline behavior.Only those anamolies are flagged which surpass a certain threshold value for any resource or give out some kind of errors.    

### Anamoly Identification Result

- Expected anomaly missed: no clear expected anomaly was missed in the provided data; the timeout records were correctly identified.
- False positive on normal behavior: none were observed in this dataset.

### Relevant log and metric details

The anomaly report is readable and explains why each record was flagged. The important signals are the combination of:

- very high latency (`response_time_ms` well above the normal range),
- elevated CPU and memory usage,
- and an `ERROR` log message indicating a timeout condition.

This is sufficient for understanding why the observation was considered abnormal.

### Limitation and possible improvement

A limitation of the current detection approach is that it relies on fixed metric thresholds and only explicitly checks for `WARNING` log severity in the detector logic. In practice, the repository data contains `ERROR` logs that are more relevant to the incident, so a possible improvement would be to include explicit error-log correlation and time-windowed incident grouping to better combine metrics and logs into a single operational signal.

This matches the assessment goal: the AIOps workflow is intended to identify operational degradation by correlating abnormal metrics and concerning log events into a detectable incident.

## Task-4: Validation of the AIOps Event Flow

The repository contains a lightweight event-streaming simulation made up of a producer, a topic, and a consumer. The purpose of the flow is to carry anomaly information from the detection step into downstream processing and reporting.

### Components and their roles

- `EventProducer`: takes a detected anomaly event and pushes it to the in-memory topic.
- `EventTopic`: is the buffer or message bus that stores the published events.
- `EventConsumer`: reads the events from the topic and delivers them for downstream use.
- `Event/message`: the structured anomaly record containing the timestamp, service, event type, reasons, and the original source telemetry.

The orchestration layer is `src/aiops_pipeline.py`, which coordinates the detection step and event publication/consumption process.

### Execution result

The repository’s event-streaming simulation was executed using the provided producer, topic, and consumer components. The verification result was:

- `records_processed= 10`
- `anomalies_detected= 2`
- `events_published= 2`
- `events_received= 2`

The two emitted anomaly events were:

1. `2026-09-20T10:05:00` - `ANOMALY` for `payment-service` with reason: `High response time`
2. `2026-09-20T10:06:00` - `ANOMALY` for `payment-service` with reasons: `High response time`, `High CPU utilization`, `High memory utilization`

These results show that:

1. An anomaly identified by the detector produces an event.
2. The event is passed to the producer.
3. The producer publishes the event to the appropriate topic.
4. The consumer receives the event from the topic.
5. The consumer processes the received event.
6. The processed event reaches the downstream AIOps component.

This confirms that an anomaly can travel through the complete event-processing pipeline in the repository’s lightweight simulation.

## Task-5: Workflow Investigation and Corrections

The assessment environment contains a small number of workflow issues that prevent the complete AIOps process from operating correctly unless they are corrected in place.

### Issue 1: Topic mismatch between producer and consumer

- Component affected: the event flow in `src/aiops_pipeline.py`
- Cause: the producer published to `service-events`, while the consumer was reading from a different topic (`anomaly-events`)
- Correction: the consumer must read from the same topic object created for the producer
- Verification: after correcting the topic assignment, the workflow produced `Events consumed: 2`, confirming that the anomaly events reached downstream processing

### Issue 2: Incorrect log severity check in anomaly detection

- Component affected: `src/anomaly_detector.py`
- Cause: the detector was checking for `WARNING` instead of the actual failure-state `ERROR` values present in the repository data
- Correction: the detector now treats `ERROR` log events as relevant anomaly signals in the same existing architecture
- Verification: the timeout events were correctly identified with an error log reason attached

### Result after correction

The corrected workflow was executed with:

```bash
cd /workspaces/github-skills-challenge
python3 src/aiops_pipeline.py
```

Observed output:

```text
==================================================
AIOps Pipeline Result
==================================================
Records processed: 10
Anomalies detected: 2
Events consumed: 2

Detected Events:

Service: payment-service
Timestamp: 2026-09-20T10:05:00
Type: ANOMALY
Reasons: High response time, Error log detected

Service: payment-service
Timestamp: 2026-09-20T10:06:00
Type: ANOMALY
Reasons: High response time, High CPU utilization, High memory utilization, Error log detected
```

This confirms that the workflow now operates correctly within the existing architecture: telemetry is processed, anomaly events are published, the consumer receives the same event stream, and the downstream AIOps layer can act on the anomaly details.

## Task-6: End-to-End Pipeline Execution

After completing the investigation and corrections, the complete AIOps workflow was executed to confirm the full data path:

`Operational Data -> Anomaly Detection -> Event -> Producer -> Topic -> Consumer -> AIOps`

### Verification criteria

The final workflow execution was checked against the following conditions:

1. Operational data is processed.
2. Anomalous behaviour is detected.
3. An anomaly event is generated.
4. The event is published.
5. The event is consumed.
6. The event is processed successfully.
7. The final output represents the detected operational issue.

### Execution command

```bash
cd /workspaces/github-skills-challenge
python3 src/aiops_pipeline.py
```

### Observed output

```text
==================================================
AIOps Pipeline Result
==================================================
Records processed: 10
Anomalies detected: 2
Events consumed: 2

Detected Events:

Service: payment-service
Timestamp: 2026-09-20T10:05:00
Type: ANOMALY
Reasons: High response time, Error log detected

Service: payment-service
Timestamp: 2026-09-20T10:06:00
Type: ANOMALY
Reasons: High response time, High CPU utilization, High memory utilization, Error log detected
```

This output confirms the complete end-to-end flow: the service telemetry was processed, anomalies were detected, event objects were created and sent through the producer/topic/consumer path, and the final AIOps result correctly represented the operational issue affecting the payment service.

## Task-7: Final Project Summary and Reproduction Guide

### 1. AIOps scenario

This repository models a payment-processing service called `payment-service`. The operational problem is that the service can slow down or fail under resource pressure, resulting in elevated latency, increased CPU and memory consumption, and timeout-related error log messages. The purpose of the AIOps workflow is to detect these signals early, correlate the relevant telemetry, emit anomaly events, and surface a likely operational incident for investigation.

### 2. Operational data description

The operational data is stored in `data/service_data.json` and contains a compact sequence of health records collected once per minute. Each record includes a timestamp, the service name, request latency in milliseconds, CPU and memory percentages, the log severity, and the message text describing the event. This data is sufficient to distinguish normal operation from degraded conditions.

### 3. Observations from the logs and metrics

The normal records show stable service behavior:

- response times remain around 120-150 ms
- CPU usage remains roughly 42-50%
- memory usage remains roughly 51-57%
- log records are `INFO` and report successful payment processing

The unusual records are the samples around `2026-09-20T10:05:00` and `2026-09-20T10:06:00`:

- response times rise to 610 ms and 640 ms
- CPU reaches 75% and 94%
- memory reaches 70% and 91%
- logs switch to `ERROR`
- messages indicate `Payment service timeout` and `Database connection timeout`

These conditions clearly represent degraded service behavior.

### 4. Anomaly-detection findings

The `AnomalyDetector` flags records when response time, CPU utilization, or memory utilization exceeds the configured thresholds, and when an `ERROR` log event is present. In this dataset, the detector identifies two abnormal observations:

- `2026-09-20T10:05:00` — `High response time`, `Error log detected`
- `2026-09-20T10:06:00` — `High response time`, `High CPU utilization`, `High memory utilization`, `Error log detected`

These anomalies correspond to the service degradation seen in the metrics and logs.

### 5. Event-processing flow

The event flow in the repository is:

`Operational Data -> Anomaly Detection -> Event -> Producer -> Topic -> Consumer -> AIOps`

The core components are:

- `AnomalyDetector`: evaluates telemetry to decide whether an event is anomalous.
- `EventProducer`: sends the generated anomaly event to the topic.
- `EventTopic`: stores the in-memory event stream.
- `EventConsumer`: reads from the topic and passes the message downstream.

### 6. Final workflow execution result

The corrected end-to-end workflow was executed with the command below:

```bash
cd /workspaces/github-skills-challenge
python3 src/aiops_pipeline.py
```

Observed output:

```text
==================================================
AIOps Pipeline Result
==================================================
Records processed: 10
Anomalies detected: 2
Events consumed: 2

Detected Events:

Service: payment-service
Timestamp: 2026-09-20T10:05:00
Type: ANOMALY
Reasons: High response time, Error log detected

Service: payment-service
Timestamp: 2026-09-20T10:06:00
Type: ANOMALY
Reasons: High response time, High CPU utilization, High memory utilization, Error log detected
```

This demonstrates the full AIOps pipeline working end-to-end: the operational data was processed, abnormal behavior was detected, anomaly events were published to the topic, the consumer received them, and the final output reflected the service incident.

### 7. Issues identified and corrected

Two issues were discovered during workflow validation:

1. Topic mismatch between producer and consumer
   - The producer and consumer were not subscribed to the same topic.
   - Corrected by ensuring the consumer reads from the same topic instance that the producer published to.

2. Incorrect log severity logic
   - The detector looked for `WARNING` instead of the actual failure-level `ERROR` messages present in the data.
   - Corrected by checking for `ERROR`, which matches the real timeout events in the dataset.

### 8. Limitation and possible improvement

The current implementation is a simple threshold-based detector. It is clear and effective for this synthetic dataset, but it may miss more complex incidents or fail to correlate multiple related anomalies over time. A better approach would be to add time-window correlation and incident grouping so related spikes and error messages are treated as one service event instead of several isolated alerts.

### 9. Reproduction steps

Another user can reproduce the demonstration by following these steps:

1. Open the repository root.
2. Confirm the project contains the required files: `data/service_data.json`, `src/anomaly_detector.py`, `src/event_producer.py`, `src/event_topic.py`, `src/event_consumer.py`, and `src/aiops_pipeline.py`.
3. Run the workflow from the project root:

```bash
cd /workspaces/github-skills-challenge
python3 src/aiops_pipeline.py
```

4. Review the console output to confirm the service processed 10 records and detected 2 anomalies.
5. If needed, validate the event stream directly by running the producer/topic/consumer sequence in the `src` directory with the repository’s operational data.

This README documents the complete scenario, the data and findings, the event flow, the corrected issues, the final execution result, and the steps needed to reproduce the demonstration.

---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

