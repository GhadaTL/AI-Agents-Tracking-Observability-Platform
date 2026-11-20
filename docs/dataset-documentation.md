# Dataset Exploration and Schema Analysis

**Dataset:**
[Exgentic/agent-llm-traces](https://huggingface.co/datasets/Exgentic/agent-llm-traces)\
**Analysis scope:** 39 Parquet shards, 1,781 session records, 38,052
spans\
**Status:** Exploration and the schema/identifier checks described in
this document have been performed. Full record-level duplicate
validation and final documentation review remain separate follow-up
checks if required by the ticket.

------------------------------------------------------------------------

## 2. Dataset overview 
### 2.1 Source

-   **Hugging Face dataset:** `Exgentic/agent-llm-traces`
-   **Format:** Parquet
-   **Number of files:** 39 shards
-   **Total reported dataset size:** approximately 938.03 MB
-   **Total records:** 1,781 top-level session records
-   **Total nested spans:** 38,052

The Parquet files are named from `train-00000-of-00039.parquet` to
`train-00038-of-00039.parquet`.

The files were located in the local Hugging Face cache during the
analysis. Their location on one machine is environment-specific and
should not be used as a permanent project path.

### 2.2 Distribution of records across files

The shard row counts were consistent:

  Shards                             Rows per shard   Number of shards   Total rows
  -------------------------------- ---------------- ------------------ ------------
  `train-00000` to `train-00025`                 46                 26        1,196
  `train-00026` to `train-00038`                 45                 13          585
  **Total**                                                     **39**    **1,781**

### 2.3 Main dataset concepts

The dataset is organized at two levels:

-   **Top-level record:** represents a session and contains metadata
    about the harness, benchmark, models, token counts, and collection
    time.
-   **Nested span:** represents an individual recorded operation within
    the session/trace. The `spans` field is a list of span objects.

In this dataset, the identifier analysis confirmed a one-to-one
relationship between top-level sessions and traces: each of the 1,781
sessions is associated with exactly one trace, and each trace is
associated with exactly one session.

### 2.4 Top-level columns

All 39 Parquet files were found to share the same eight top-level
columns.

  -----------------------------------------------------------------------
  Column                  Description             Observed structure
  ----------------------- ----------------------- -----------------------
  `harness`               Agent harness or        Categorical/string
                          execution environment   

  `benchmark`             Benchmark or task suite Categorical/string

  `models`                Model(s) associated     List
                          with the session        

  `max_tokens`            Maximum token setting   Numeric
                          recorded for the        
                          session                 

  `total_tokens`          Total token count       Numeric
                          recorded for the        
                          session                 

  `session_id`            Identifier of the       String
                          session                 

  `spans`                 Nested list of recorded List of structs
                          operations              

  `collected_at`          Collection timestamp    String
  -----------------------------------------------------------------------

The exact semantic interpretation and units of token-related fields
should be confirmed against the dataset documentation before downstream
business logic relies on them.

## 3. Schema analysis --- MLOP-6

### 3.1 Nested `spans` structure

The `spans` field is a list of structured span records. Each span
contains 14 fields:

  Field                   Description
  ----------------------- ---------------------------------------------
  `span_id`               Identifier of the span
  `name`                  Span name
  `kind`                  OpenTelemetry span kind
  `start_time`            Span start timestamp, stored as a string
  `end_time`              Span end timestamp, stored as a string
  `attributes`            Nested GenAI attributes
  `resource_attributes`   Metadata about the instrumented service/SDK
  `status`                Span status information
  `type`                  Operation type
  `harness`               Harness associated with the span
  `benchmark`             Benchmark associated with the span
  `models`                Model list associated with the span
  `session_id`            Session identifier repeated at span level
  `trace_id`              Trace identifier

### 3.2 Nested `attributes`

The `attributes` struct contains 10 fields:

  ----------------------------------------------------------------------------------
  Attribute                          Type observed           Description
  ---------------------------------- ----------------------- -----------------------
  `gen_ai.operation.name`            String                  GenAI operation name

  `gen_ai.request.model`             String                  Model requested

  `gen_ai.response.model`            String                  Model reported in the
                                                             response

  `gen_ai.usage.input_tokens`        Int64                   Input token usage

  `gen_ai.usage.output_tokens`       Int64                   Output token usage

  `gen_ai.response.id`               String                  Response identifier

  `gen_ai.response.finish_reasons`   List of strings         Finish reason(s)

  `gen_ai.input.messages`            String                  Serialized input
                                                             messages

  `gen_ai.output.messages`           String                  Serialized output
                                                             messages

  `gen_ai.tool.definitions`          String                  Serialized tool
                                                             definitions
  ----------------------------------------------------------------------------------

The message and tool-definition fields are serialized as strings in the
observed schema. In particular, `gen_ai.output.messages` contains
JSON-encoded message data and must be parsed before its nested parts can
be analyzed.

### 3.3 `resource_attributes`

The `resource_attributes` struct contains five fields:

-   `telemetry.sdk.language`
-   `telemetry.sdk.name`
-   `telemetry.sdk.version`
-   `service.name`
-   `service.version`

These fields describe the telemetry SDK and the instrumented service.

### 3.4 `status`

The `status` struct contains:

-   `code` --- Int64
-   `message` --- String

The observed status-code distribution is shown in Section 5.3. The
interpretation of numeric status codes should follow the dataset's
OpenTelemetry conventions; under the OpenTelemetry status-code
definition, `1` means `OK` and `2` means `ERROR`.

## 4. Attribute completeness and missing values

The attribute-presence analysis was performed across all **38,052
spans**.

### 4.1 Attribute presence

  Attribute                            Spans with a value   Percentage
  ---------------------------------- -------------------- ------------
  `gen_ai.operation.name`                          38,052      100.00%
  `gen_ai.request.model`                           38,052      100.00%
  `gen_ai.response.model`                          38,052      100.00%
  `gen_ai.usage.input_tokens`                      38,052      100.00%
  `gen_ai.usage.output_tokens`                     38,052      100.00%
  `gen_ai.input.messages`                          38,052      100.00%
  `gen_ai.response.id`                             36,713       96.48%
  `gen_ai.response.finish_reasons`                 36,713       96.48%
  `gen_ai.output.messages`                         36,713       96.48%
  `gen_ai.tool.definitions`                        28,008       73.60%

### 4.2 Null values

  Attribute                            Null values   Percentage
  ---------------------------------- ------------- ------------
  `gen_ai.output.messages`                   1,339        3.52%
  `gen_ai.response.finish_reasons`           1,339        3.52%
  `gen_ai.response.id`                       1,339        3.52%
  `gen_ai.tool.definitions`                 10,044       26.40%

No null values were observed for the other six attributes.

### 4.3 Empty values

The checks found no empty strings, empty lists, or empty dictionaries in
the analyzed attribute values.

### 4.4 Interpretation and data-engineering implications

-   The six attributes present in all spans are strong candidates for
    core fields in the normalized span representation.
-   The response-related attributes are nullable and should remain
    nullable in the Silver layer.
-   `gen_ai.tool.definitions` is absent in 26.40% of spans. This should
    not automatically be interpreted as a failed tool call: tool
    definitions describe tools made available to the model, not the
    actual tool calls made.
-   Do not automatically replace missing values with `0` or empty
    strings. Preserve nulls unless a documented transformation rule
    justifies a replacement.
-   `gen_ai.input.messages`, `gen_ai.output.messages`, and tool
    definitions may contain user content or sensitive data. Avoid
    unnecessary payload logging or exposure; apply appropriate access,
    retention, and minimization controls.

## 5. Categorical and operational distributions

### 5.1 Session-level distributions

The following counts are based on **1,781 top-level session records**.

**Harness**

  Harness                              Sessions
  ---------------------------------- ----------
  `claude_code`                             839
  `tool_calling`                            431
  `smolagents_code`                         315
  `openai_solo`                             114
  `tool_calling_with_shortlisting`           82

**Benchmark**

  Benchmark            Sessions
  ------------------ ----------
  `tau2_retail`             469
  `appworld`                406
  `swebench`                391
  `tau2_airline`            196
  `tau2_telecom`            186
  `browsecompplus`          133

**Models**

The model counts below represent the number of sessions in which each
model string appeared. A session may contain more than one model, so
these counts are not mutually exclusive and do not have to sum to 1,781.

  Model string                     Sessions containing the model
  ------------------------------ -------------------------------
  `openai/Azure/gpt-4.1`                                     550
  `aws/claude-opus-4-5`                                      509
  `Azure/gpt-4.1`                                            365
  `gcp/gemini-3-pro-preview`                                 361
  `azure/Kimi-K2.5`                                          158
  `azure/DeepSeek-V3.2`                                      128
  `Azure/gpt-5.2-2025-12-11`                                  37
  `openai/azure/Kimi-K2.5`                                    29
  `openai/azure/DeepSeek-V3.2`                                18
  `gpt-5.2-2025-12-11`                                        15

**Normalization caution:** similar-looking model strings, such as
`openai/Azure/gpt-4.1` and `Azure/gpt-4.1`, should not be merged without
validating whether they refer to the same provider/model identity.

### 5.2 Span-level distributions

The following counts are based on **38,052 spans**.

  Field           Value                   Count   Percentage
  --------------- -------------------- -------- ------------
  `kind`          `SPAN_KIND_CLIENT`     38,052      100.00%
  `type`          `llm_call`             38,052      100.00%
  `status.code`   `1`                    36,713       96.48%
  `status.code`   `2`                     1,339        3.52%

Under the OpenTelemetry status-code convention, code `1` is `OK` and
code `2` is `ERROR`. Confirm that the dataset follows this convention
before using these values as definitive success/failure labels in
metrics or ML features.

### 5.3 Finish reasons

The finish-reason counts were:

  Finish reason                                     Count
  ------------------------------------------ ------------
  `stop`                                           25,317
  `tool_calls`                                     11,394
  `content_filter`                                      2
  **Total non-null finish-reason entries**     **36,713**

The total is 36,713 because 1,339 spans have a null
`gen_ai.response.finish_reasons` value.

## 6. Tool-call analysis

### 6.1 Extraction method

`gen_ai.output.messages` is stored as a JSON string. The JSON was
parsed, then each message's `parts` list was inspected for objects whose
`type` is `tool_call`.

This structure matters: searching only at the top level of the decoded
JSON for an object with `type = "tool_call"` would miss the tool calls
nested inside `message.parts`.

### 6.2 Global results

The extraction was performed across all 39 Parquet shards.

  Metric                                                     Result
  -------------------------------------------- --------------------
  Files analyzed                                                 39
  Total spans                                                38,052
  Spans containing at least one tool call                    25,565
  Total tool calls extracted                                 33,390
  Percentage of spans containing tool calls                  67.19%
  Average calls per span that contains calls     Approximately 1.31

A tool call in an output message indicates that the model requested a
tool call. It does not, by itself, prove that the tool was successfully
executed.

### 6.3 Most frequent tool names

  Tool name                                      Call count
  -------------------------------------------- ------------
  `mcp__environment__bash`                            8,551
  `mcp__environment__search`                          3,186
  `mcp__environment__message`                         1,685
  `bash`                                              1,548
  `spotify__show_song`                                1,088
  `TodoWrite`                                         1,034
  `simple_note__search_notes`                           883
  `mcp__environment__simple_note__show_note`            862
  `simple_note__show_note`                              837
  `venmo__show_transactions`                            760

Tool names with and without prefixes should not be merged automatically.
Their identity and semantics should be checked first. Tool names and
arguments can also reveal sensitive capabilities or data; this report
records selected names and counts only, not tool arguments or message
payloads.

## 7. Identifier and relationship validation

These checks were run across the complete dataset, not just a sample.

### 7.1 Session identifiers

  Check                              Result
  -------------------------------- --------
  Total sessions                      1,781
  Missing `session_id` values             0
  Unique `session_id` values          1,781
  Duplicated `session_id` values          0
  Duplicate occurrences                   0

Every top-level session record has a non-null and unique `session_id`.

### 7.2 Span and trace identifiers

  Check                                                  Result
  ---------------------------------------------------- --------
  Total spans                                            38,052
  Missing `span_id` values                                    0
  Unique `span_id` values globally                       33,524
  `span_id` values appearing more than once globally      4,528
  Duplicate occurrences globally                          4,528
  Missing `trace_id` values                                   0
  Unique `trace_id` values                                1,781

A globally repeated `span_id` is not automatically an error. In
OpenTelemetry, a span ID is expected to be unique within its trace, not
necessarily across the entire dataset.

### 7.3 Span-ID collision analysis

A second check grouped spans by `span_id` and examined the associated
`trace_id` values.

  Check                                       Result
  ----------------------------------------- --------
  Span IDs appearing in multiple traces        4,528
  Span IDs repeated within the same trace          0

**Interpretation:** the 4,528 globally repeated span IDs occur across
different traces. No repeated `span_id` was found within the same trace.
Therefore, these global repetitions should not be treated as duplicate
spans or removed solely on the basis of `span_id`.

For identification and duplicate checks, use the composite key
**(`trace_id`, `span_id`)** rather than `span_id` alone.

### 7.4 Session-to-trace relationship

  Check                                  Result
  ------------------------------------ --------
  Sessions analyzed                       1,781
  Sessions linked to multiple traces          0
  Traces linked to multiple sessions          0

The observed relationship is one-to-one: each session is associated with
exactly one trace, and each trace is associated with exactly one
session.

### 7.5 Top-level to span-level session consistency

The `session_id` stored in each nested span was compared with the
`session_id` of its containing top-level record.

  Check                              Result
  -------------------------------- --------
  Total spans checked                38,052
  Matching session IDs               38,052
  Mismatching session IDs                 0
  Missing span-level session IDs          0

All 38,052 span-level session identifiers match their parent session
identifier. No missing or mismatching values were detected.

## 8. Consolidated findings

The exploration and schema analysis established the following:

1.  The dataset consists of 39 Parquet shards containing 1,781 session
    records and 38,052 nested spans.
2.  All shards share the same eight top-level columns and the same
    nested span schema.
3.  The span schema contains identifiers, timestamps, GenAI attributes,
    resource attributes, status information, and session/trace metadata.
4.  Six of the ten GenAI attributes are populated for all spans. Four
    attributes contain nulls, which should be preserved unless a
    justified transformation rule is defined.
5.  Approximately 67.19% of spans contain at least one extracted tool
    call.
6.  The dataset contains six benchmark categories and five main harness
    categories in the observed session distribution.
7.  All session IDs and trace IDs are present. Session IDs are unique,
    and the observed session-to-trace relationship is one-to-one.
8.  All span IDs and trace IDs are present. Repeated span IDs occur
    across different traces, not within the same trace.
9.  All span-level session IDs match their parent session IDs.

Overall, the checks performed show consistent identifier relationships
and a stable schema across the 39 analyzed files. The null values and
serialized message fields remain important considerations for downstream
data processing.




## Appendix A --- Key metrics at a glance

  Metric                                                Value
  ----------------------------------------- -----------------
  Parquet shards                                           39
  Top-level session records                             1,781
  Total spans                                          38,052
  Unique session IDs                                    1,781
  Unique trace IDs                                      1,781
  Unique span IDs globally                             33,524
  Span IDs repeated across traces                       4,528
  Span IDs repeated within the same trace                   0
  Spans with tool calls                                25,565
  Tool calls extracted                                 33,390
  Spans with matching parent session ID       38,052 / 38,052
