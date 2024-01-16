# LogQL Demystified: A Comprehensive Guide to Log Query Language

LogQL is Grafana Loki’s PromQL-inspired query language. Queries act as if they are a distributed `grep` to aggregate log sources. LogQL uses labels and operators for filtering.

There are two types of LogQL queries:
- `Log queries` return the contents of log lines.
- `Metric queries` extend log queries to calculate values based on query results.


## Operators in LogQL
In order to understand Queries , we need to understand Operators first.

### Binary Operators:
1. **Arithmetic Binary Operators:**
- `+`(Addition), `-` (Subtraction), `*` (Multiplication), `/` (Division)
 
2. **Comparison Binary Operators:**
- `==` (Equal), `!=` (Not Equal), `<` (Less Than), `>` (Greater Than), `<=` (Less Than or Equal), `>=` (Greater Than or Equal)
 
3. **Logical Binary Operators:**
- `and` (Logical AND), `or` (Logical OR), `unless` (Logical NOT)
 
4. **Set Binary Operators:**
- `=~` (Regex Match), `!~` (Negative Regex Match)
 
5. **Mathematical Binary Operators:**
-  `^` (Exponentiation), `%` (Modulo)

### Order of operations
When chaining or combining operators, you have to consider operator precedence: Generally, you can assume regular mathematical convention with operators on the same precedence level being left-associative.

### Keywords on and ignoring
The `ignoring` keyword causes specified labels to be ignored during matching. The syntax:
```text
<vector expr> <bin-op> ignoring(<labels>) <vector expr>
```

## Log queries

Everything starts with a log stream pipeline that collects logs from various sources and stores them into a logstream storage solution like Loki.

Once the log streams are stored, then we want to be able to consume and transform the collected data to build dashboards or alerts for your project.
 
LogQL allows you to filter, transform, and extract data as metrics. Once you have metrics, you will be able to use any functions from PromQL to aggregate them. After applying a filter and metric, it can also return a log stream.

LogQL is composed of a {Stream selector} and a log pipeline where each step of your pipeline is separated by a vertical bar  `|` .

<div style="text-align:center;">
  <img src="./images/query_components.png" alt="Query Components" style="width:90%;">
</div>


###  The log agent collector
When collecting logs, our log agent collector adds context to our log stream like the pod name, the service name, etc. So the log stream selector allows us to filter our logs based on the labels available in our log stream.


## The log pipeline

The log pipeline efficiently processes and filters the log stream using label matching operators. It comprises key components:

1. **Line filter:** Selects specific log entries based on defined criteria.
2. **Parser:** Structures log entries for better interpretation and analysis.
3. **Label filter:** Filters logs based on labels, refining the log selection.
4. **Line format:** Defines the format for log entries.
5. **Labels format:** Specifies the format for labels associated with logs.
6. **Unwrap (for metrics):** Extracts relevant metrics from logs.

Exploring each component individually offers a comprehensive understanding of their roles in optimizing log stream management.

### Line filter

The line filter acts like a `grep` for aggregated logs, searching log line content. Operators include:

- `|=`: Log line contains string.
- `!=`: Log line does not contain string.
- `|~`: Log line contains a match to the regular expression.
- `!~`: Log line does not contain a match to the regular expression.

Example:

```text
{container="frontend"} |= "error"
{cluster="us-central-1"} |= "error" != "timeout"
```

This example filters logs to only include those with errors, showcasing the flexibility of the line filter in refining log selections.

### Parser expression

The parser expression transforms our log stream, extracting labels using various functions:

- **JSON:** Parses JSON-formatted logs.
- **Logfmt:** Extracts keys and values from logfmt format lines.
- **Parser:** Custom parsing based on specified rules.
- **Unpack:** Extracts data from binary formats.
- **Regexpp:** Applies regular expressions for parsing.

Example:

```text
{container="frontend"} |= "error" | JSON
```

For JSON-formatted logs:

```json
{
  "pod.name": { "id": "deded" },
  "namespace": "test"
}
```

Applying the JSON parser exposes new labels in the transformed log stream:

- pod_name_id = "deded"
- namespace = test

Parameters in JSON allow selective extraction of desired labels.

Logfmt example:

```text
at=info method=GET path=/
host=mutelight.org fwd="123.443.21.212" status=200 bytes=1653
```

Transformed to:

- At = "info"
- method = GET
- path = /
- host = mutelight.org

### Pattern parser

The `Pattern` function is a robust parser tool for explicitly extracting fields from log lines. In this example log stream:

```text
192.176.12.1[10/Jun/2021:09:14:29 +0000] "GET /api/plugins/versioncheck HTTP/1.1" 200 2 "-" "Go-http-client/2.0" "13.76.247.102, 34.120.177.193" "TLSv1.2" "US" ""
```

You can use the following expression to parse the log line:

```text
<ip> - - <_> "<method> <uri> <_>" <status> <size> <_> "<agent>" <_> "<ip_list>" "<protocol>" "<country>" <_>
```

Using `<_>` indicates you're not interested in keeping a specific label for those fields. The `Pattern` function empowers precise extraction of information, enhancing log analysis.


### Regexp parser
Regexp is similar to the pattern, but you can specify the expected format utilizing your regexp regular expression.

### Label filter

After parsing the log stream and introducing new labels, you can apply refined label filtering. Building upon the previous expression:

```text
{container="frontend"} |= "error" | JSON
```

You can add:

```text
{container="frontend"} 
|= "error" 
| JSON
| duration > 1m and 
bytes_consumed > 20MB
```

This extended expression demonstrates a specific interest in logs with an error, where the duration exceeds 1 minute, and the bytes consumed surpass 20MB. The ability to filter based on these newly parsed labels enhances log analysis precision.

### Line Format Expression

The `Line format` expression empowers you to reshape your log content by displaying specific labels. For instance:

```text
{container="frontend"}
| logfmt 
| line_format "{{.ip}} 
{{.status}} 
{{div .duration 1000}}"
```

In this example, the log content is reformatted to showcase only the "ip," "status," and the duration converted to seconds. This capability offers flexibility in presenting log data in a concise and tailored manner for analysis and interpretation.

### Labels Format Expression

The `Label format` function provides the ability to rename, modify, and add labels to the modified log stream. Subsequently, Metric Queries are employed to extract log streams and metrics from the logs.

## Metric Queries

Metric queries apply a function to log query results and return a range vector. Two types of aggregators exist: Log range aggregation and Unwrapped range aggregation.

### Log range aggregation

Similar to Prometheus, a range aggregation is a query followed by a duration. Various supported functions include:

- `rate(log-range)`: calculates entries per second.
- `count_over_time(log-range)`: counts entries for each log stream within the given range.
- `bytes_rate(log-range)`: calculates bytes per second for each stream.
- `bytes_over_time(log-range)`: counts the bytes used by each log stream for a given range.
- `absent_over_time(log-range)`: returns an empty vector if the range vector has any elements and a 1-element vector with the value 1 if the range vector has no elements. (Useful for alerting when no time series and logs stream exist for a label combination for a certain time.)

Example:

```text
sum by (host) (
rate(
 {job="mysql"} 
 |= "error" != "timeout" 
 | JSON 
 | duration > 10s 
[1m])
)
```

In this example, logs are split by the host, focusing on MySQL-related jobs, excluding timeouts. Parsed with JSON for additional labels, and filtered for durations above 10 seconds. This powerful combination of label manipulation and metric queries enhances the observability of log data.

### Unwrap range aggregations

`Unwrap` specifies which labels expose metrics, supporting various functions for unwrapped ranges:

- `rate(unwrapped-range)`: calculates per-second rate of all values in the specified interval.
- `sum_over_time(unwrapped-range)`: the sum of all values in the specified interval.
- `avg_over_time(unwrapped-range)`: the average value of all points in the specified interval.
- `max_over_time(unwrapped-range)`: the maximum value of all points in the specified interval.
- `min_over_time(unwrapped-range)`: the minimum value of all points in the specified interval.
- `first_over_time(unwrapped-range)`: the first value of all points in the specified interval.
- `last_over_time(unwrapped-range)`: the last value of all points in the specified interval.
- `stdvar_over_time(unwrapped-range)`: the population standard variance of the values in the specified interval.
- `stddev_over_time(unwrapped-range)`: the population standard deviation of the values in the specified interval.
- `quantile_over_time(scalar,unwrapped-range)`: the φ-quantile (0 ≤ φ ≤ 1) of the values in the specified interval.

Example:

```text
quantile_over_time(0.99,
 {cluster="ops-tools1",container="ingress-nginx"}
 | JSON
 | __error__ = ""
 | unwrap request_time [1m])) by (path)
```

In this example, we calculate the 99th percentile, filtering specific labels for ops-tools1 and the container. After parsing with JSON and removing potential errors, we unwrap request_time above 1m. This flexibility in unwrapping ranges enhances the precision of metric extraction.

## Acknowledgment 

Special thanks to **Henrik Rexed** for her valuable guidance and teachings. You can find him on [Is it Observable](https://www.youtube.com/c/IsitObservable).