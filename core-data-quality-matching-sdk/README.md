
# Matching SDK MCP Server

> 📦 **Download:** [core-data-quality-matching-sdk.beta.v1.0](https://github.com/PreciselyData/precisely-mcp-servers/releases/tag/core-data-quality-matching-sdk.beta.v1.0)

> ⚠️ **BETA RELEASE** — This is a beta release of the Matching SDK MCP Server. Feedback and bug reports are welcome. APIs and behaviors may change as we refine the implementation based on usage.

## Summary

The Matching SDK MCP Server exposes data matching capabilities as tools for AI models and development environments. It provides programmatic access to the Precisely Matching SDK through a stdio-based interface, enabling configuration management, data inspection, and distributed matching job execution across local or remote Spark clusters.

### Prerequisites

Before running the MCP server, ensure you have:

- **Java 17 or higher** installed ([Download](https://www.oracle.com/java/technologies/downloads/#java17))
    - Verify: `java -version`

- **Apache Spark 3.3+** (required only for job execution)
    - Required for the `embedded` or `submit` backends
    - Not required for the `remote` backend (jobs run on a remote cluster)
    - Verify: `spark-submit --version`

- **Writable staging directory** (required for `remote` backend)
    - Used to stage job inputs, specs, and results
    - Must be visible to both the MCP host and the Spark cluster (local path or mount)


## Beta Release Scope

**Committed Tools (Shipped):**
- `inspect` — Runtime readiness, bounded data preview, workflow guidance
- `manage_config` — Configuration discovery, validation, creation, retrieval
- `run_match` — Intra/inter matching execution with task lifecycle and idempotency
- `run_match_task` — Async job status, results, and cancellation

**Committed Features:**
- Three execution backends: `embedded` (local Spark), `submit` (spark-submit), `remote` (Spark REST API)
- Deterministic specialist orchestration and validation-gating
- Docker Compose deployment support

**Deferred from Beta (Planned for Future Release):**
- `name_tools` — Parser and variant name management

## Available Tools

The MCP server exposes the following tools:

- **manage_config** — Discover, inspect, validate, and create matching configuration artifacts
    - Supported actions:
        - `discover_config_types` — Lists available configuration families (match_key, match_rule)
        - `list_algorithms` — Lists available algorithms within a config type with pagination support
        - `list_samples` — Lists sample configurations for experimentation and learning
        - `validate` — Validates a configuration against schema requirements and reports issues
        - `get` — Retrieves a stored configuration by ID/File
        - `create_match_key` — Creates a new match_key configuration from raw JSON, samples, or structured intent
        - `create_match_rule` — Creates a new match_rule configuration from raw JSON, samples, or structured intent

- **inspect** — Analyze runtime health, perform bounded data preview, and access workflow guidance
    - Supported actions:
        - `spark_status` — Reports Spark availability and runtime health (host-local or Docker-backed)
        - `preview_data` — Inspects CSV, JSON, or Parquet files without executing matching, returning schema and bounded row samples
            - Parameters: `inputDescriptorJson` (required, JSON with sourceType, path/attachmentId, formatHint), `maxRows` (optional, default 100), `maxBytes` (optional, default 1MB)
            - Returns: schema, row samples, truncation flag, and diagnostics
        - `workflow_guide` — Retrieves the deterministic orchestration workflow as markdown, providing agents with authoritative guidance on workflow semantics and interaction patterns

- **run_match** — Execute matching jobs using the configured Spark backend
    - Supported actions:
        - `intra` — Intra matching with a single dataset
        - `inter` — Inter matching with two datasets
    - Supported backends:
        - `remote` — Submits jobs to a Spark master via REST API (requires MCP_SPARK_MASTER_URL)
        - `embedded` — Uses local JVM Spark runtime (default when SPARK_HOME is set)
        - `submit` — Launches spark-submit from the MCP host (requires MCP_SPARK_RUNNER_JAR)

- **run_match_task** — Query or manage long-running match executions
    - Supported operations:
        - `status` — Polls the current status of a submitted job
        - `result` — Retrieves the result manifest once the job completes
        - `cancel` — Requests cancellation of a running job

See tools manifest for complete details.

## Installation

### Step 1: Download the JAR

The MCP server is distributed as a single executable JAR file: `matching-sdk-mcp.jar`

If you plan to use the `submit` or `remote` backend, distribute the Spark runner JAR alongside it:

- `matching-sdk-mcp.jar` - MCP server process
- `lib/mcp-spark-runner.jar` - Spark application JAR used by `spark-submit`

Distribution output layout is:

```text
matching-sdk-mcp.zip
  matching-sdk-mcp.jar
  lib/
    mcp-spark-runner.jar
```

Choose a location to store the MCP server (examples below):

If you use the `submit` backend, copy the runner JAR too:

**Linux/macOS:**
```bash
mkdir -p ~/.local/share/mcp-servers/matching-sdk/lib
cp matching-sdk-mcp.jar ~/.local/share/mcp-servers/matching-sdk/
cp lib/mcp-spark-runner.jar ~/.local/share/mcp-servers/matching-sdk/lib/
```

**Windows (PowerShell):**
```powershell
$MCPPath = "$env:APPDATA\mcp-servers"
New-Item -ItemType Directory -Force -Path "$MCPPath\lib"
Copy-Item matching-sdk-mcp.jar -Destination $MCPPath
Copy-Item lib\mcp-spark-runner.jar -Destination "$MCPPath\lib"
```

## Running the Server

The JAR file is a complete, self-contained executable with all dependencies embedded.

### With environment variables
```bash
export QUARKUS_LOG_LEVEL=DEBUG
export SPARK_HOME=/opt/spark
java -jar matching-sdk-mcp.jar
```

**Windows (PowerShell):**
```powershell
$env:QUARKUS_LOG_LEVEL = "DEBUG"
$env:SPARK_HOME = "C:\path\to\spark"
java -jar matching-sdk-mcp.jar
```

### Backend modes

The MCP server can run matching jobs through three Spark execution paths:

- `embedded` - uses the local JVM Spark runtime
- `submit` - launches `spark-submit` from the MCP host
- `remote` - submits jobs to a Spark master through the Spark REST API on port `6066`

When `MCP_SPARK_MASTER_URL` is set, the remote backend takes precedence over embedded and submit backends.

#### Remote backend

The remote backend uses Spark master REST submissions instead of launching `spark-submit` locally.

Beta release note:

- The Spark master REST API port `6066` must be exposed and reachable from the MCP server
- This beta flow assumes deployment inside a trusted network boundary such as a VPN
- There is currently no authentication layer added by the MCP server for Spark REST submissions, so `6066` should not be exposed to the public Internet

Flow:

1. The MCP server converts `MCP_SPARK_MASTER_URL` from `spark://host:7077` to the Spark REST endpoint `http://host:6066`
2. It stages the job spec, inputs, and output locations into a shared root visible to both the MCP host and the Spark cluster
3. It submits `mcp-spark-runner.jar` to the master using `/v1/submissions/create`
4. It polls the Spark submission status through the REST API and reads the result manifest from the staged output directory

This mode is intended for Docker or remote-cluster scenarios where the MCP server cannot directly run the Spark driver in the same environment as the cluster.

## Docker Deployment

For containerized environments, the MCP server can be deployed alongside a local Spark cluster using Docker Compose.

### Prerequisites

- Docker and Docker Compose installed

### Quick Start with Docker Compose

A pre-configured `docker-compose.yaml` is available in the source repository. It orchestrates:

- **MCP Server** - Matching SDK MCP container
- **Spark Master** - Cluster master node on port 7077 and REST API on port 6066
- **Spark Worker** - Optionally worker node for job execution
- **Shared Volume** - `mcp-output` directory for staging and results

### Configuration for Docker

The `docker-compose.yaml` automatically configures:

- `MCP_SPARK_MASTER_URL=spark://spark-master:7077` - Remote backend targeting the cluster
- `MCP_SHARED_ROOT=/mcp-shared` - Container-mounted staging directory
- `MCP_RUNNER_JAR_EXECUTOR=/mcp-dist/lib/mcp-spark-runner.jar` - Runner JAR path inside container

#### docker-compose.yaml Sample

A minimal `docker-compose.yaml` for local development:

```yaml
services:
  spark-master:
    image: spark:3.5.7-scala2.12-java17-ubuntu
    container_name: spark-master
    hostname: spark-master
    ports:
      - "6066:6066"
      - "7077:7077"
      - "8081:8081"
    volumes:
      - ~/.local/share/mcp-servers/matching-sdk:/mcp-dist:ro
      - ./mcp-output:/mcp-shared
    environment:
      SPARK_HOME: "/opt/spark"
      SPARK_MASTER_HOST: "spark-master"
      SPARK_MASTER_PORT: "7077"
      SPARK_MASTER_WEBUI_PORT: "8081"
      SPARK_MASTER_OPTS: "-Dspark.master.rest.enabled=true -Dspark.master.rest.host=0.0.0.0 -Dspark.master.rest.port=6066"
    command: /opt/spark/sbin/start-master.sh

  spark-worker:
    image: spark:3.5.7-scala2.12-java17-ubuntu
    hostname: spark-worker
    depends_on:
      - spark-master
    volumes:
      - ~/.local/share/mcp-servers/matching-sdk:/mcp-dist:ro
      - ./mcp-output:/mcp-shared
    environment:
      SPARK_MASTER_URL: "spark://spark-master:7077"
    command: /opt/spark/sbin/start-worker.sh spark://spark-master:7077

  matching-sdk-mcp:
    image: eclipse-temurin:17-jre
    container_name: matching-sdk-mcp
    volumes:
      - ~/.local/share/mcp-servers/matching-sdk:/mcp-dist:ro
      - ./mcp-output:/mcp-shared
    environment:
      MCP_SPARK_MASTER_URL: "spark://spark-master:7077"
      MCP_SHARED_ROOT: "/mcp-shared"
      MCP_RUNNER_JAR_EXECUTOR: "/mcp-dist/lib/mcp-spark-runner.jar"
    command: java -jar /mcp-dist/matching-sdk-mcp.jar
    depends_on:
      - spark-master
      - spark-worker
```

**Setup instructions:**

1. Complete the Installation section above to extract and install the MCP server files to `~/.local/share/mcp-servers/matching-sdk/`

2. Create a `docker-compose.yaml` in your working directory (use the sample above)

3. Start services:
```bash
mkdir -p ./mcp-output  # Shared staging directory
docker-compose up -d
```

4. Verify services are running:
```bash
docker-compose ps
docker-compose logs -f matching-sdk-mcp
```

5. Access the Spark cluster:
    - **Spark Master UI:** http://localhost:8081

### Integration with MCP Clients

To connect your MCP client (Claude Desktop, VS Code) to the Docker-deployed server:

**VS Code MCP Configuration (mcp.json):**
```json
{
  "matching-sdk-mcp-docker": {
    "type": "stdio",
    "command": "docker",
    "args": [
      "compose",
      "-f",
      "/path/to/docker-compose.yaml",
      "run",
      "--rm",
      "-T",
      "matching-sdk-mcp"
    ],
    "env": {
      "MCP_SPARK_MASTER_URL": "spark://spark-master:7077",
      "MCP_SHARED_ROOT": "/mcp-shared"
    },
    "alwaysAllow": ["inspect", "match_config"]
  }
}
```

This configuration runs the MCP server inside the Docker Compose environment on-demand, with automatic networking and volume access to the Spark cluster.

## Configuration

Set environment variables before running:

- QUARKUS_LOG_LEVEL - Logging level (default: INFO)
    - Possible values: OFF, ERROR, WARN, INFO, DEBUG, TRACE

- SPARK_HOME - Path to Apache Spark installation (optional)
    - If set, enables Docker Spark detection

- MCP_SPARK_MASTER_URL - Enables the remote backend and identifies the Spark master
    - Format: `spark://hostname:7077`
    - The MCP server derives the REST submission endpoint as `http://hostname:6066`

- MCP_SPARK_DRIVER_MASTER_URL - Optional driver-facing Spark master URL for the remote backend
    - Use this when the REST submission endpoint and the driver runtime see the cluster under different hostnames
    - Example: host submits to `spark://localhost:7077`, but the driver inside Docker must use `spark://spark-master:7077`

- MCP_SHARED_ROOT - Host-local writable staging directory for remote execution artifacts
    - Stores staged inputs, job specs, and outputs that the MCP server can read back after completion

- MCP_SHARED_ROOT_EXECUTOR - Executor-visible path for the same shared root
    - Example: host path `C:\matching-sdk\mcp-output` mapped as `/mcp-shared` inside containers

- MCP_SPARK_RUNNER_JAR - Host-local path to `mcp-spark-runner.jar` for the `submit` backend
    - Example: `C:\Users\you\AppData\Roaming\mcp-servers\lib\mcp-spark-runner.jar`

- MCP_RUNNER_JAR_EXECUTOR - Executor-visible runner JAR path for remote REST execution
    - The `submit` backend will also accept this value only when the same path exists on the local host

- MCP_SPARK_DRIVER_EXTRA_JAVA_OPTIONS - Optional value passed as `--conf spark.driver.extraJavaOptions=...` for the `submit` backend

- MCP_SPARK_EXECUTOR_EXTRA_JAVA_OPTIONS - Optional value passed as `--conf spark.executor.extraJavaOptions=...` for the `submit` backend

- SPARK_DRIVER_JAVA_OPTIONS and SPARK_EXECUTOR_JAVA_OPTIONS
    - Used as fallback sources for the same submit-time Spark conf if the `MCP_` prefix variants are not set

### Example: Running with debug logging
```bash
export QUARKUS_LOG_LEVEL=DEBUG
export SPARK_HOME=/opt/spark
java -jar matching-sdk-mcp.jar
```

### Example: Submit backend with explicit runner JAR
```bash
export SPARK_HOME=/opt/spark
export MCP_SPARK_RUNNER_JAR="$HOME/.local/share/mcp-servers/lib/mcp-spark-runner.jar"
java -jar matching-sdk-mcp.jar
```

**Windows (PowerShell):**
```powershell
$env:SPARK_HOME = "C:\spark"
$env:MCP_SPARK_RUNNER_JAR = "$env:APPDATA\mcp-servers\lib\mcp-spark-runner.jar"
java -jar matching-sdk-mcp.jar
```


## MCP Client Integration

### Cursor or Claude/VS Code with MCP Extension

The MCP configuration file location depends on your client:

**Claude Desktop:** `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows)

**VS Code with MCP Extension:** Typically in a `.mcp` or `.mcp-servers` config directory

### Configuration Examples

**Linux/macOS Configuration (mcp.json or claude_desktop_config.json):**
```json
{
  "matching-mcp": {
    "type": "stdio",
    "command": "java",
    "args": ["-jar", "~/.local/share/mcp-servers/matching-sdk/matching-sdk-mcp.jar"],
    "env": {
      "SPARK_HOME": "/opt/spark",
      "MCP_SPARK_RUNNER_JAR": "~/.local/share/mcp-servers/matching-sdk/lib/mcp-spark-runner.jar"
    }
  }
}
```

**Linux/macOS Configuration for remote backend:**
```json
{
  "matching-mcp": {
    "type": "stdio",
    "command": "java",
    "args": ["-jar", "~/.local/share/mcp-servers/matching-sdk/matching-sdk-mcp.jar"],
    "env": {
      "MCP_SPARK_MASTER_URL": "spark://localhost:7077",
      "MCP_SPARK_DRIVER_MASTER_URL": "spark://spark-master:7077",
      "MCP_SHARED_ROOT": "/absolute/path/to/mcp-output",
      "MCP_SHARED_ROOT_EXECUTOR": "/mcp-shared",
      "MCP_RUNNER_JAR_EXECUTOR": "/mcp-dist/lib/mcp-spark-runner.jar"
    }
  }
}
```


**Windows Configuration for remote backend:**
```json
{
  "matching-mcp": {
    "type": "stdio",
    "command": "java",
    "args": ["-jar", "%APPDATA%\\mcp-servers\\matching-sdk-mcp.jar"],
    "env": {
      "MCP_SPARK_MASTER_URL": "spark://localhost:7077",
      "MCP_SPARK_DRIVER_MASTER_URL": "spark://spark-master:7077",
      "MCP_SHARED_ROOT": "C:\\Temp\\mcp-output",
      "MCP_SHARED_ROOT_EXECUTOR": "/mcp-shared",
      "MCP_RUNNER_JAR_EXECUTOR": "/mcp-dist/lib/mcp-spark-runner.jar",
      "QUARKUS_LOG_LEVEL": "INFO"
    }
  }
}
```

### After Configuration

1. Save the configuration file
2. Restart Claude Desktop or VS Code
3. The MCP server should appear in the available tools list
4. Test it by asking Claude to analyze data or inspect runtime status


## Troubleshooting

### Server fails to start
1. Verify Java 17+ is installed: `java -version`
2. Check SPARK_HOME is correct (if using Spark features)
4. Run with verbose output: `java -jar matching-sdk-mcp.jar`

### Remote backend submission fails
1. Verify `MCP_SPARK_MASTER_URL` points to a reachable Spark master
2. Verify the Spark master REST API is reachable on port `6066`
3. Verify `MCP_SHARED_ROOT` is writable on the host
4. Verify `MCP_SHARED_ROOT_EXECUTOR` and `MCP_RUNNER_JAR_EXECUTOR` match the paths visible inside the Spark cluster
5. If the driver runs in a different network context, set `MCP_SPARK_DRIVER_MASTER_URL` to the hostname the driver should use to find the master

### ClassNotFoundException or missing classes
All dependencies are embedded in the JAR. If you encounter missing classes:
1. Verify you're running the complete `matching-sdk-mcp.jar` (check file size, ~60-100MB)
2. Check JAR integrity: `jar -tf matching-sdk-mcp.jar | head`
3. Ensure no partial downloads

If you use the `submit` backend:
1. Verify `mcp-spark-runner.jar` is present and readable
2. Set `MCP_SPARK_RUNNER_JAR` to the host-local runner JAR path
3. Only use `MCP_RUNNER_JAR_EXECUTOR` for submit when that exact path also exists on the host machine

### Java 17 serialization or module-access failures
If Spark jobs fail with Java 17 reflective-access or serialization issues, pass module-open flags to both driver and executor.

For the `submit` backend, the MCP server maps these environment variables to Spark submit conf:

- `MCP_SPARK_DRIVER_EXTRA_JAVA_OPTIONS` -> `spark.driver.extraJavaOptions`
- `MCP_SPARK_EXECUTOR_EXTRA_JAVA_OPTIONS` -> `spark.executor.extraJavaOptions`
- `SPARK_DRIVER_JAVA_OPTIONS` -> fallback for driver when `MCP_SPARK_DRIVER_EXTRA_JAVA_OPTIONS` is unset
- `SPARK_EXECUTOR_JAVA_OPTIONS` -> fallback for executor when `MCP_SPARK_EXECUTOR_EXTRA_JAVA_OPTIONS` is unset

Example:

```bash
export MCP_SPARK_DRIVER_EXTRA_JAVA_OPTIONS="--add-opens java.base/sun.nio.ch=ALL-UNNAMED --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.lang.invoke=ALL-UNNAMED --add-opens java.base/java.util=ALL-UNNAMED"
export MCP_SPARK_EXECUTOR_EXTRA_JAVA_OPTIONS="--add-opens java.base/sun.nio.ch=ALL-UNNAMED --add-opens java.base/java.lang=ALL-UNNAMED --add-opens java.base/java.lang.invoke=ALL-UNNAMED --add-opens java.base/java.util=ALL-UNNAMED"
```

These become:

```text
--conf spark.driver.extraJavaOptions=...
--conf spark.executor.extraJavaOptions=...
```

For embedded or externally managed Spark runtimes, set the equivalent Spark conf or container JVM options in the runtime that actually launches the driver and executors.

### Backend selection issues

If the wrong backend is selected or rejected:

1. **`remote` backend rejected when `SPARK_MASTER_URL` is set:** The server forbids explicit `embedded` or `submit` when `SPARK_MASTER_URL` is active. Remove `SPARK_MASTER_URL` to allow embedded/submit, or remove conflicting backend override.
2. **`embedded`/`submit` rejected with no SPARK_HOME:** Set `SPARK_HOME` to enable embedded/submit, or set `MCP_SPARK_MASTER_URL` for remote backend.
3. **Verify active backend:** Check the `inspect.spark_status` response; `sparkSource` indicates the resolved source (HOST_LOCAL, DOCKER_LOCAL, NONE for degraded).

### Configuration validation errors (`manage_config`)

If validation fails with structured issues:

1. **Missing `availableColumns`:** If column mapping fails, provide `availableColumns` in the validate request to enable per-column checks.
2. **Schema structure errors:** Ensure JSON matches expected shape (e.g., `matchKeys` array for match_key, `rules` array for match_rule). Use `list_samples` to see valid structures.
3. **Idempotency conflicts:** Reusing an `idempotencyToken` with different `draftJson` returns a conflict error. Use a unique token or omit idempotency for non-retryable operations.

### Input file issues (`run_match` and `preview_data`)

1. **File not found (MCP-002):** Verify the `host_path` exists and is readable; use `preview_data` first to confirm accessibility.
2. **Permission denied:** Ensure the MCP process has read access to the dataset file.
3. **Attachment materialization failed (MCP-007):** If using `attachment` source type, verify the attachment ID is correct and material.

### Preview issues (`inspect.preview_data`)

1. **Unsupported format (MCP-003):** Only CSV, JSON, and Parquet are supported. Check `formatHint` parameter.
2. **Parquet preview requires Spark (MCP-004):** Parquet preview is Spark-gated. Ensure Spark is available or use CSV/JSON formats.
3. **Preview truncated:** If `truncated: true`, increase `maxRows` (up to 10000) or `maxBytes` (default 1MB) to capture more data.

### Common error codes

| Code | Name | Typical Cause | Retryable |
| --- | --- | --- | --- |
| MCP-001 | VALIDATION_ERROR | Invalid input schema or conflicting params | No |
| MCP-002 | FILE_NOT_FOUND | Dataset or config file not accessible | No |
| MCP-003 | UNSUPPORTED_FORMAT | CSV/JSON/Parquet expected | No |
| MCP-004 | SPARK_UNAVAILABLE | Spark required but not configured | No |
| MCP-005 | RUNTIME_DEPENDENCY_UNAVAILABLE | Transient service outage | Yes |
| MCP-006 | INPUT_TOO_LARGE | Dataset exceeds size limits | No |
| MCP-007 | ATTACHMENT_MATERIALIZATION_FAILED | Attachment upload or staging failed | No |
| MCP-999 | INTERNAL_ERROR | Unexpected server error | Yes |

### Trace IDs and diagnostics

All error responses include a `traceId` for support correlation. Logs are written to stderr; enable `QUARKUS_LOG_LEVEL=DEBUG` to capture per-action diagnostics. Include the `traceId` when reporting issues.
