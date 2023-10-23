---
title: My first OpenTelemetry exporter in Go
description: Understanding how the OpenTelemetry collector works by creating my first exporter
slug: first-opentelemetry-exporter
date: 2023-10-23 00:00:00+0000
original: true
categories:
    - Development
tags:
    - Go
    - OpenTelemetry
---

Ever since I discovered [OpenTelemetry](https://opentelemetry.io/) for application monitoring and the flexibility of the [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/), I've been sharing its wonders with several of my clients – and they've had great experiences too.

Having used the Collector for some time, my curiosity about its inner workings grew. That's when I decided to create my own exporter to get a deeper understanding. To keep the focus on understanding the core mechanics, I crafted a basic exporter that doesn't process the received metrics, traces, or logs in any way. This project was a perfect chance for me to explore two things: diving into [Go](https://go.dev/), which was new for me, and uncovering more about the OpenTelemetry Collector's operations, especially how it sends metrics to its configured destinations.

For those keen to see the code, you can find it all [here](https://github.com/droosma/first-opentelemetry-exporter).

## Building a Custom Collector

If you're looking to run, test, and debug your custom exporter, you'll need to construct a custom collector. The OpenTelemetry documentation offers a [fantastic guide](https://opentelemetry.io/docs/collector/custom-collector/) on this. But, given that this was my maiden voyage into Go, and my increasing frustration with altering my development environment every time I experiment, I opted for the [devcontainer](https://containers.dev/) approach.

The fundamental `devcontainer.json` setup is rather straightforward:

```JSON
{
  "name": "Go",
  "image": "mcr.microsoft.com/devcontainers/go:0-1-bullseye",
  "forwardPorts": [4317,4318],
  "postCreateCommand": "bash .devcontainer/postCreateCommand.sh"
}
```

You'll notice I employ the `postCreateCommand` to invoke an external script. This is because, to build the custom collector, certain dependencies must be met. I faced issues running the commands within `postCreateCommand.sh` directly from `devcontainer.json`. To minimize troubleshooting, I chose this strategy. One drawback is that whenever Visual Studio Code detects modifications to `devcontainer.json`, it prompts to rebuild and restart the container. With dependencies in a separate file, manual container rebuilding is necessary.

We also unblock the default OpenTelemetry ports 4317 (gRPS) and 4318 (http), enabling data transmission from our application to the custom collector.

`postCreateCommand.sh` will install the `builder` and `delve` commands, essential for building the custom collector and facilitating [debugging](#debugging-the-exporter).

Next, a manifest file is required to guide the builder. I saved this as `otelcol-builder.yaml` in my repository's root.

```yaml
dist:
  name: otelcol-custom
  description: Local OpenTelemetry Collector binary
  output_path: /tmp/dist
  otelcol_version: 0.88.0
exporters:
  - gomod: go.opentelemetry.io/collector/exporter/debugexporter v0.88.0
receivers:
  - gomod: go.opentelemetry.io/collector/receiver/otlpreceiver v0.88.0
```

> *Note: Working with a `devcontainer` for this project raised an `output_path` configuration issue. To modify it, refer to the `output_path` section below. [^1]*

Now, build the custom collector using:

```bash
builder --config=otelcol-builder.yaml
```

This produces a `otelcol-custom` binary in the `/tmp/dist` directory. Running this binary initiates the custom collector. However, it requires a configuration file first. I named this `config.yaml` and stored it next to the `otelcol-builder.yaml` in my repository's root. Starting with just the `debug` exporter makes it simpler to confirm the custom collector's functionality.

```yaml
receivers:
  otlp:
    protocols:
      http:
      grpc:
exporters:
  debug:
    verbosity: detailed
service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: 
        - debug
    metrics:
      receivers: [otlp]
      exporters: 
        - debug
    logs:
      receivers: [otlp]
      exporters: 
        - debug
```

Finally, with the configuration set, execute the following to see logs, metrics, and traces displayed in the console:

```bash
/tmp/dist/otelcol-custom --config=config.yaml
```

## The exporter Code

Now that we've set up a working custom collector, it's time to dive into our exporter. I began by creating a new folder named `emptyexporter` in my repository's root. Inside this folder, I created a `go.mod` file. This file informs Go that we're dealing with a module and facilitates the import of other necessary modules. The foundational `go.mod` file is:

```go
module github.com/droosma/emptyexporter

go 1.20

require (
  go.opentelemetry.io/collector/component v0.88.0
  go.opentelemetry.io/collector/exporter v0.88.0
  go.opentelemetry.io/collector/pdata v1.0.0-rcv0017
)
```

Typically, the module name mirrors the repository name. While this isn't a strict requirement, it simplifies module imports in other projects. In this instance, the name isn't an actual repository name. But since we won't be publishing this module, that's fine. However, ensure that the repo name aligns with the folder on your local machine, as Go uses this to locate the module.

The `require` section lists the modules we need. The `go.opentelemetry.io/collector/component` module is essential for configuration creation, the `go.opentelemetry.io/collector/exporter` module aids in exporter creation, and the `go.opentelemetry.io/collector/pdata` module handles the various telemetry categories or [signals](https://opentelemetry.io/docs/concepts/signals/).

Next, I crafted an `exporter.go` file within the `emptyexporter` folder. This file houses our exporter code. Given that our exporter won't be particularly complex, the code remains fairly straightforward:

```go
package emptyexporter

import (
  "context"

  "go.opentelemetry.io/collector/pdata/plog"
  "go.opentelemetry.io/collector/pdata/pmetric"
  "go.opentelemetry.io/collector/pdata/ptrace"
)

type emptyexporter struct {
}

func NewEmptyexporter() *emptyexporter {
  return &emptyexporter{}
}

func (s *emptyexporter) pushLogs(_ context.Context, ld plog.Logs) error {
  return nil
}

func (s *emptyexporter) pushMetrics(ctx context.Context, md pmetric.Metrics) error {
  return nil
}

func (s *emptyexporter) pushTraces(_ context.Context, td ptrace.Traces) error {
  return nil
}
```

The `emptyexporter` struct defines our exporter. The functions `pushLogs`, `pushMetrics`, and `pushTraces` will be invoked by the collector upon receipt of logs, metrics, and traces, respectively. As we aren't processing the received telemetry, these functions simply return `nil`.

The function `NewEmptyexporter` serves as a factory function, generating a new instance of the `emptyexporter` struct.

With the exporter code ready, it's time to register it with the collector. This requires a `factory.go` file in the `emptyexporter` directory. This file contains the registration code:

```go
package emptyexporter

import (
  "context"

  "go.opentelemetry.io/collector/component"
  "go.opentelemetry.io/collector/exporter"
  "go.opentelemetry.io/collector/exporter/exporterhelper"
)

const (
  typeStr = "emptyexporter"
)

func NewFactory() exporter.Factory {
  return exporter.NewFactory(
    typeStr,
    createDefaultConfig,
    exporter.WithTraces(createTracesExporter, component.StabilityLevelDevelopment),
    exporter.WithMetrics(createMetricsExporter, component.StabilityLevelDevelopment),
    exporter.WithLogs(createLogsExporter, component.StabilityLevelDevelopment),
  )
}

func createTracesExporter(
  ctx context.Context,
  set exporter.CreateSettings,
  config component.Config) (exporter.Traces, error) {

  fg := config.(*Config)
  s := NewEmptyexporter()
  return exporterhelper.NewTracesExporter(ctx, set, cfg, s.pushTraces)
}

func createMetricsExporter(
  ctx context.Context,
  set exporter.CreateSettings,
  config component.Config) (exporter.Metrics, error) {

  cfg := config.(*Config)
  s := NewEmptyexporter()
  return exporterhelper.NewMetricsExporter(ctx, set, cfg, s.pushMetrics)
}

func createLogsExporter(
  ctx context.Context,
  set exporter.CreateSettings,
  config component.Config) (exporter.Logs, error) {

  cfg := config.(*Config)
  s := NewEmptyexporter()
  return exporterhelper.NewLogsExporter(ctx, set, cfg, s.pushLogs)
}

type Config struct {
}

func createDefaultConfig() component.Config {
  return &Config{}
}
```

The function `NewFactory` is our factory function, crafting a new instance of the `exporter.Factory` struct, complete with the `typeStr` as the exporter type and the `createDefaultConfig` function to produce the exporter's default configuration. Since our configuration is currently non-existent, it merely returns an empty struct. The `NewFactory` function also references the `exporter.WithTraces`, `exporter.WithMetrics`, and `exporter.WithLogs` functions to register the exporter with the collector for traces, metrics, and logs, respectively. Each of these functions, in turn, calls the respective internal factory methods, `createTracesExporter`, `createMetricsExporter`, and `createLogsExporter`.

The only task left is to refresh the `otelcol-builder.yaml` file to encompass the `emptyexporter` module. Within the exporter section, add:

```yaml
exporters:
  - gomod: "github.com/droosma/emptyexporter v0.0.1"
    path: emptyexporter
```

While a module version is mandatory, any version will suffice since we aren't publishing this module. The `path` denotes the module's path on your machine. Remember, even if you specify the module's path, Go will still search for the module using the repository name identical to the module name.

Lastly, we'll update the `config.yaml` file to include the `emptyexporter` exporter:

```yaml
service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: 
        - emptyexporter
        - debug
    metrics:
      receivers: [otlp]
      exporters: 
        - emptyexporter
        - debug
    logs:
      receivers: [otlp]
      exporters: 
        - emptyexporter
        - debug
```

After rebuilding the custom collector and initiating it once more, we're set to test our exporter.

## Debugging the exporter

Now the whole purpose of building your own version of the collector is to be able to debug your exporter. Now that we have a something to debug, let's see how we can do that.

The first thing you should do is add `debug_compilation: true` to the `dist` section in the `otelcol-builder.yaml` file. This will include the debug symbols in the binary when you build your collector, which will allow us to debug the exporter.

```yaml
dist:
  debug_compilation: true
```

After some searching I found that you can debug your go application by using the [delve](https://github.com/go-delve/delve) as I have found a lot of resources on how to debug go applications using delve, I decided to go with this approach.

Now we can start the custom collector in debug mode by running the following command.

```bash
dlv --listen=:2345 --headless=true --api-version=2 --accept-multiclient --log exec /tmp/dist/otelcol-custom -- --config=config.yaml
```

Next we will need to create a `launch.json` file in the `.vscode` folder. This file will tell Visual Studio Code how to attach to `delve` and debug the exporter.

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Connect to server",
            "type": "go",
            "request": "attach",
            "mode": "remote",
            "port": 2345,
            "host": "127.0.0.1",
            "apiVersion": 2,
            "showLog": true
        }
    ]
}
```

Now all that's left is to go to the `Run and Debug` tab in Visual Studio Code and select `Connect to server` from the dropdown and click the play button. This will attach Visual Studio Code to the custom collector and you can now set breakpoints in your exporter on on `pushLogs`, `pushMetrics` or `pushTraces` functions and see the data trickle in.

## Conclusion

You should now have a working custom collector with a working exporter that you can debug. ofcourse this is just a walking skeleton of an exporter, but it should give you a good starting point to build your own exporter.
for inspiration on what you can do, check out the receivers, exporters and processors in the wild, from the [OpenTelemtry Collector](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter) and [OpenTelemetry Collector Contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter).

If you like to have an example of the data that you can expect to receive in your exporter, you can check out [example JSON](https://opentelemetry.io/docs/specs/otel/protocol/file-exporter/#examples)

[^1]: changing the `output_path`

    As I was annoyed that the `output_path` pointed to a location inside of the container, I tried to change it to a location on my local machine by changing it to `/workspaces/opentelemetry-embedding-exporter/otelcol-custom` but this did not work. Executing the build command resulted in the following error.

    ```bash
    Error: failed to compile the OpenTelemetry Collector distribution: exit status 1. Output:
    error obtaining VCS status: exit status 128
        Use -buildvcs=false to disable VCS stamping.
    ```

    I'm sure it should be possible, but to again avoid a massive jack that needs shaving, I accepted it and moved on. 
    If you know how to do this, please let me know.
