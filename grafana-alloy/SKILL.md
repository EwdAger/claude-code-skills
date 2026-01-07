---
name: grafana-alloy
description: Grafana Alloy usage and configuration guide (OpenTelemetry Collector + Prometheus pipelines). Use when writing Alloy configs, building metrics/logs/traces pipelines, configuring prometheus.scrape/remote_write, Loki log collection and processing, OTLP receive/export, or working with import/declare/livedebugging/exports/forward_to and log processing stages.
---

# Grafana Alloy Skill

A Grafana Alloy configuration and usage guide covering syntax, component wiring, common pipelines (metrics/logs/traces), and best practices.

## Core Concepts

### Components and Pipelines

- **Component block**: `<component.type> "<label>" { ... }`
- **Data flow**: Use `forward_to` to connect upstream output to downstream `receiver`
- **Export reference**: `component.label.export`, e.g. `local.file.api_key.content`
- **Custom component**: `declare` + `argument` + `export`

### Common Utility Blocks

- `livedebugging { enabled = true }`
- `import.file` to include external config
- `file.path_join` to join paths
- `sys.env` / `env` for environment variables
- `constants.hostname` for built-in constants

### Common Wiring Pattern

```
source -> process -> write/export
```

## Quick Start

### Finding and Running Alloy

**Common Alloy binary locations:**
- Custom installation: `./alloy` (current directory)
- System-wide: `/usr/local/bin/alloy` or `/usr/bin/alloy`
- Package manager: `which alloy` to find it

**Basic commands:**

```bash
# Format and validate configuration syntax
alloy fmt config.alloy

# Run a configuration
alloy run config.alloy

# Check Alloy version
alloy --version
```

**Setting required environment variables before running:**

```bash
# Export environment variables that your config needs
export SOURCE_FILE_DIR=/path/to/config/directory
export LOKI_ENDPOINT="http://localhost:3100/loki/api/v1/push"
export IPADDR="192.168.1.100"

# Then run Alloy
alloy run config.alloy
```

### Config Syntax Example: Components and References

```alloy
local.file "api_key" {
  filename  = "/var/data/secrets/api-key"
  is_secret = true
}

prometheus.remote_write "prod" {
  endpoint {
    url = "https://prod:9090/api/v1/write"
    basic_auth {
      username = "admin"
      password = local.file.api_key.content
    }
  }
}

discovery.kubernetes "pods" {
  role = "pod"
}

prometheus.scrape "default" {
  targets    = discovery.kubernetes.pods.targets
  forward_to = [prometheus.remote_write.prod.receiver]
}
```

## Logs Pipeline (Loki)

Use `local.file_match` + `loki.source.file` to collect logs, `loki.process` to parse them, then `loki.write` to send to Loki.

```alloy
local.file_match "applogs" {
  path_targets = [{"__path__" = "/tmp/app-logs/app.log"}]
}

loki.source.file "local_files" {
  targets    = local.file_match.applogs.targets
  forward_to = [loki.process.add_new_label.receiver]
}

loki.process "add_new_label" {
  stage.logfmt {
    mapping = {
      "extracted_level" = "level",
    }
  }

  stage.labels {
    values = {
      "level" = "extracted_level",
    }
  }

  forward_to = [loki.write.local_loki.receiver]
}

loki.write "local_loki" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

### Common Log Processing Stages

```alloy
loki.process "log_filter" {
  stage.multiline {
    firstline     = "^\\d{4}-\\d{2}-\\d{2}"
    max_wait_time = "10s"
  }

  stage.drop {
    expression = "DEBUG"
  }

  stage.regex {
    expression = "command\\s+(?P<db>\\w+)\\.(?P<collection>[^\\s]+)"
  }

  stage.structured_metadata {
    values = {
      db = "db",
    }
  }

  stage.static_labels {
    values = {
      job = "app",
    }
  }

  stage.labels {
    values = {
      collection = "collection",
    }
  }
}
```

### Log Collection Options

- `local.file_match.sync_period` controls scan frequency
- `loki.source.file.tail_from_end` controls whether to start tailing at the file end

## Metrics Pipeline (Prometheus)

Use `prometheus.scrape` to scrape targets and `prometheus.remote_write` to write to a remote Prometheus endpoint.

```alloy
prometheus.scrape "metrics_test_local_agent" {
  targets = [{
    __address__ = "127.0.0.1:12345",
    cluster     = "localhost",
  }]
  job_name        = "local-agent"
  scrape_interval = "15s"
  forward_to      = [prometheus.remote_write.metrics_test.receiver]
}

prometheus.remote_write "metrics_test" {
  endpoint {
    name = "test-remote"
    url  = "https://prometheus-us-central1.grafana.net/api/prom/push"

    basic_auth {
      username = "<USERNAME>"
      password = "<PASSWORD>"
    }
  }
}
```

## Traces Pipeline (OpenTelemetry)

Use `otelcol.receiver.otlp` to receive OTLP, `otelcol.processor.batch` to batch, and `otelcol.exporter.otlp` to export.

```alloy
otelcol.receiver.otlp "default" {
  grpc {
    include_metadata = true
  }
  http {
    include_metadata = true
  }

  output {
    traces = [otelcol.processor.batch.default.input]
  }
}

otelcol.processor.batch "default" {
  timeout         = "20s"
  send_batch_size = 10000

  output {
    traces = [otelcol.exporter.otlp.default_0.input]
  }
}

otelcol.exporter.otlp "default_0" {
  client {
    endpoint = "tempo:4317"
  }
}
```

## Common Patterns and Tips

### 1) Inject secrets via file or environment

```alloy
local.file "api_key" {
  filename  = "/var/data/secrets/api-key"
  is_secret = true
}

prometheus.remote_write "metrics_service" {
  endpoint {
    url = sys.env("GCLOUD_HOSTED_METRICS_URL")
    basic_auth {
      username = sys.env("GCLOUD_HOSTED_METRICS_ID")
      password = local.file.api_key.content
    }
  }
}
```

### 2) Export reference rules

- Syntax: `<component>.<label>.<export>`
- Example: `local.file.api_key.content`

### 3) Import external config and use env vars

```alloy
livedebugging {
  enabled = true
}

import.file "mongo" {
  filename = file.path_join(sys.env("SOURCE_FILE_DIR"), "mongo.alloy")
}

loki.write "default" {
  endpoint {
    url = sys.env("LOKI_ENDPOINT")
  }
}
```

## Reusable Components Checklist

- **Collect**: `prometheus.scrape`, `loki.source.file`, `otelcol.receiver.otlp`
- **Process**: `loki.process`, `otelcol.processor.batch`
- **Export**: `prometheus.remote_write`, `loki.write`, `otelcol.exporter.otlp`
- **Discover/Utility**: `discovery.kubernetes`, `local.file`, `local.file_match`

## Custom Components (Optional)

Use `declare` to encapsulate reusable logic.

```alloy
declare "add" {
  argument "a" { }
  argument "b" { }

  export "sum" {
    value = argument.a.value + argument.b.value
  }
}

add "example" {
  a = 15
  b = 17
}
```

### Custom Component Args and Reuse Example

```alloy
declare "log_filter" {
  argument "write_to" {
    optional = false
  }

  loki.relabel "add_label" {
    forward_to = argument.write_to.value
    rule {
      target_label = "ipaddr"
      replacement  = env("IPADDR")
    }
  }
}
```

## Configuration Validation and Debugging

### 1) Validate configuration syntax

Before running Alloy, always validate your configuration:

```bash
# Format and check syntax - will show errors if invalid
alloy fmt your-config.alloy

# The command outputs the formatted config if successful
# Or shows syntax errors if the config is invalid
```

### 2) Common validation errors and fixes

**Missing environment variables:**
```
Error: failed to build component: environment variable "LOKI_ENDPOINT" not found
```
Fix: Export the required environment variable before running
```bash
export LOKI_ENDPOINT="http://localhost:3100/loki/api/v1/push"
```

**File not found:**
```
Error: failed to read file: no such file or directory
```
Fix: Ensure `SOURCE_FILE_DIR` points to the correct directory with all imported files

**Invalid regex pattern:**
```
Error: invalid regex expression
```
Fix: Escape special characters properly (use `\\` for backslash in regex)

### 3) Testing configuration step-by-step

1. **Validate syntax first:**
   ```bash
   alloy fmt main.alloy
   ```

2. **Check all imported files exist:**
   ```bash
   ls -la $SOURCE_FILE_DIR/*.alloy
   ```

3. **Verify environment variables:**
   ```bash
   echo $LOKI_ENDPOINT
   echo $SOURCE_FILE_DIR
   echo $IPADDR
   ```

4. **Run with livedebugging enabled:**
   ```alloy
   livedebugging {
     enabled = true
   }
   ```

5. **Check log files exist and are readable:**
   ```bash
   ls -la /path/to/your/log/files
   ```

### 4) Environment variables checklist

Common environment variables used in Alloy configs:
- `SOURCE_FILE_DIR` - Directory containing imported .alloy files
- `LOKI_ENDPOINT` - Loki push endpoint URL
- `IPADDR` - IP address for labeling
- `GCLOUD_HOSTED_METRICS_URL` - Cloud metrics endpoint
- `GCLOUD_HOSTED_METRICS_ID` - Cloud metrics credentials

**Example setup script:**
```bash
#!/bin/bash
# Setup environment for Alloy
export SOURCE_FILE_DIR=/sf/data/local/monitor
export LOKI_ENDPOINT="http://localhost:3100/loki/api/v1/push"
export IPADDR=$(hostname -I | awk '{print $1}')

# Validate all configs
alloy fmt $SOURCE_FILE_DIR/main.alloy

# Run Alloy
alloy run $SOURCE_FILE_DIR/main.alloy
```
