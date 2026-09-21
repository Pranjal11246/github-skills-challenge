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

I validated the actual event flow using the repository’s provided components in the expected runtime context (`src` directory), and the result was:

- `records_processed= 10`
- `anomalies_detected= 2`
- `events_published= 2`
- `events_received= 2`

The two emitted anomaly events were:

1. `2026-09-20T10:05:00` - `ANOMALY` for `payment-service` with reason: `High response time`
2. `2026-09-20T10:06:00` - `ANOMALY` for `payment-service` with reasons: `High response time`, `High CPU utilization`, `High memory utilization`

These results show that:

1. An anomaly identified by the detector produces an event.
2. That event is passed to the producer.
3. The producer publishes it to the topic.
4. The consumer reads it from the same topic.
5. The event is processed and returned to the downstream logic.
6. The downstream AIOps pipeline therefore receives a usable anomaly event for further reporting or action.

This confirms that the event-processing path works as designed within the repository’s lightweight simulation.

---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

