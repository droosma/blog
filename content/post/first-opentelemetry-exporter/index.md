---
title: My first OpenTelemetry exporter in Go
description: Understanding how the OpenTelemetry collector works by creating my first exporter
slug: first-opentelemetry-exporter
date: 2023-10-24 00:00:00+0000
original: true
image: cover.png
categories:
    - Development
tags:
    - Go
    - OpenTelemetry
---

Ever since I discovered [OpenTelemetry](https://opentelemetry.io/) for application monitoring and the flexibility of the [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/), I've been sharing its wonders with several of my clients – and they've had great experiences too.

Having used the Collector for some time, my curiosity about its inner workings grew. That's when I decided to create my own exporter to get a deeper understanding. To keep the focus on understanding the core mechanics, I crafted a basic exporter that doesn't process the received metrics, traces, or logs in any way. This project was a perfect chance for me to explore two things: diving into [Go](https://go.dev/), which was new for me, and uncovering more about the OpenTelemetry Collector's operations, especially how it sends metrics to its configured destinations.

For those keen to see the code, you can find it all [here](https://github.com/droosma/first-opentelemetry-exporter).

## Development Environment Setup

Given the fact that we are going to be developing in Go and given my increasing frustration with having to subject my machine to a bunch of new tools every time I experiment with new tech, I have started favoring the use of [devcontainers](https://containers.dev/). This approach allows me to keep my machine clean and tidy while still having all the tools I need for development. If you're not familiar with devcontainers, I highly recommend you check them out.

To get the environment setup the `devcontainer.json` setup is rather straightforward:

```JSON
{
  "name": "Go",
  "image": "mcr.microsoft.com/devcontainers/go:0-1-bullseye",
  "forwardPorts": [4317,4318],
  "postCreateCommand": "bash .devcontainer/postCreateCommand.sh"
}
```

You'll notice I employ the `postCreateCommand` to invoke an external script. This is because, to create our collector exporter we need certain tools installed other than Go. 
The `postCreateCommand.sh` file contains the following:

```bash
go install go.opentelemetry.io/collector/cmd/builder@latest
go install github.com/go-delve/delve/cmd/dlv@latest
```

The script should install the `builder` and `delve` commands. these are essential for building the custom collector and facilitating [debugging](#debugging-the-exporter).

I faced issues running these commands directly from `devcontainer.json` inside the `postCreateCommand`. To minimize troubleshooting, I chose this strategy. One drawback is that whenever Visual Studio Code detects modifications to `devcontainer.json`, it prompts to rebuild and restart the container. With dependencies in a separate file, manual container rebuilding is necessary, but given that these are all the tools we need, it's not a big deal.

In the `forwardPorts` you will see I added ports 4317 (gRPS) and 4318 (http), as I wanted to use the oltp protocol for data transmission I added these ports to enable data transmission from our application to the custom collector.

## Building a Custom Collector

To be able to run our custom exporter we will need to build a custom version of the OpenTelemetry Collector. This will allow us to run, test and debug the collector and the exporter that is part of it.
The OpenTelemetry documentation offers a [fantastic guide](https://opentelemetry.io/docs/collector/custom-collector/) on this. But, as said, I prefer the `devcontainer` approach.

OpenTelemetry offer a tool that allows us to easily build a custom collector. This tool is called [`builder`](https://github.com/open-telemetry/opentelemetry-collector/tree/main/cmd/builder) and if you have followed along with the `devcontainer` setup, you should have it installed already.

The builder uses a manifest file to know what to build. I saved this as OpenTelemetry collector builder: `otelcol-builder.yaml` in my repository's root.

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

in this file, we define the name of the binary we want to build, the description, the output path, and the version of the collector we want to use. We also define the exporters and receivers we want to include in our custom collector. For now, we only need the [`debugexporter`](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter/debugexporter) a OpenTelemetry published exporter that writes the received telemetry to the console, and the [`otlpreceiver`](https://github.com/open-telemetry/opentelemetry-collector/tree/main/receiver/otlpreceiver) the published oltp protocol implementation allowing us to receive telemetry.

> **Note**: Working with a `devcontainer` for this project raised an `output_path` configuration issue. To modify it, refer to the `output_path` section below. [^1]

Now, build the custom collector using:

```bash
builder --config=otelcol-builder.yaml
```

the output should look something like this:

```bash
vscode ➜ /workspaces/first-opentelemetry-exporter $ builder --config=otelcol-builder.yaml
2023-10-24T20:21:13.915Z        INFO    internal/command.go:121 OpenTelemetry Collector Builder {"version": "dev", "date": "unknown"}
2023-10-24T20:21:13.918Z        INFO    internal/command.go:157 Using config file       {"path": "otelcol-builder.yaml"}
2023-10-24T20:21:13.918Z        INFO    builder/config.go:108   Using go        {"go-executable": "/usr/local/go/bin/go"}
2023-10-24T20:21:13.919Z        INFO    builder/main.go:69      Sources created {"path": "/tmp/dist"}
2023-10-24T20:21:14.112Z        INFO    builder/main.go:121     Getting go modules
2023-10-24T20:21:14.194Z        INFO    builder/main.go:80      Compiling
2023-10-24T20:21:14.194Z        INFO    builder/main.go:86      Debug compilation is enabled, the debug symbols will be left on the resulting binary
2023-10-24T20:21:14.506Z        INFO    builder/main.go:102     Compiled        {"binary": "/tmp/dist/otelcol-custom"}
```

And the result should be a `otelcol-custom` binary in the `/tmp/dist` directory. Running this binary initiates the custom collector. However, at this point it's just like any other OpenTelemetry Collector and it requires a configuration file in order to run. I named this `config.yaml` and stored it next to the `otelcol-builder.yaml` in my repository's root. Starting with just the `debug` exporter makes it simpler to confirm the custom collector's functionality. The skeleton `config.yaml` file is as follows:

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

The output of that should look something like this:

```bash	
vscode ➜ /workspaces/first-opentelemetry-exporter $ /tmp/dist/otelcol-custom --config=config.yaml
2023-10-24T20:24:44.149Z        info    service@v0.88.0/telemetry.go:84 Setting up own telemetry...
2023-10-24T20:24:44.150Z        info    service@v0.88.0/telemetry.go:201        Serving Prometheus metrics      {"address": ":8888", "level": "Basic"}
2023-10-24T20:24:44.150Z        info    exporter@v0.88.0/exporter.go:275        Development component. May change in the future.        {"kind": "exporter", "data_type": "metrics", "name": "debug"}
2023-10-24T20:24:44.150Z        info    exporter@v0.88.0/exporter.go:275        Development component. May change in the future.        {"kind": "exporter", "data_type": "logs", "name": "debug"}
2023-10-24T20:24:44.150Z        info    exporter@v0.88.0/exporter.go:275        Development component. May change in the future.        {"kind": "exporter", "data_type": "traces", "name": "debug"}
2023-10-24T20:24:44.152Z        info    service@v0.88.0/service.go:143  Starting otelcol-custom...      {"Version": "1.0.0", "NumCPU": 20}
2023-10-24T20:24:44.152Z        info    extensions/extensions.go:33     Starting extensions...
2023-10-24T20:24:44.152Z        warn    internal@v0.88.0/warning.go:40  Using the 0.0.0.0 address exposes this server to every network interface, which may facilitate Denial of Service attacks        {"kind": "receiver", "name": "otlp", "data_type": "logs", "documentation": "https://github.com/open-telemetry/opentelemetry-collector/blob/main/docs/security-best-practices.md#safeguards-against-denial-of-service-attacks"}
2023-10-24T20:24:44.152Z        info    otlpreceiver@v0.88.0/otlp.go:83 Starting GRPC server    {"kind": "receiver", "name": "otlp", "data_type": "logs", "endpoint": "0.0.0.0:4317"}
2023-10-24T20:24:44.152Z        warn    internal@v0.88.0/warning.go:40  Using the 0.0.0.0 address exposes this server to every network interface, which may facilitate Denial of Service attacks        {"kind": "receiver", "name": "otlp", "data_type": "logs", "documentation": "https://github.com/open-telemetry/opentelemetry-collector/blob/main/docs/security-best-practices.md#safeguards-against-denial-of-service-attacks"}
2023-10-24T20:24:44.152Z        info    otlpreceiver@v0.88.0/otlp.go:101        Starting HTTP server    {"kind": "receiver", "name": "otlp", "data_type": "logs", "endpoint": "0.0.0.0:4318"}
2023-10-24T20:24:44.152Z        info    service@v0.88.0/service.go:169  Everything is ready. Begin running and processing data.
```

## Creating the exporter

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

The only task left is to change the `otelcol-builder.yaml` file to include our newly created `emptyexporter` module. Within the exporter section, add:

```yaml
exporters:
  - gomod: "github.com/droosma/emptyexporter v0.0.1"
    path: emptyexporter
```

While a module version is mandatory, any version will suffice since we aren't publishing this module. The `path` denotes the module's path on your machine. Remember, even if you specify the module's path, Go will still search for the module using the repository name identical to the module name.

Lastly, we'll update the `config.yaml` file to include the `emptyexporter` exporter:

```yaml
exporters:
  emptyexporter:
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

As you can see we need to add the `emptyexporter` exporter to the `exporters` section and add it to the relevant signal type pipelines, in my case `traces`, `metrics`, and `logs`.

Running the custom collector with the updated configuration file should now display the following:

```bash
vscode ➜ /workspaces/first-opentelemetry-exporter $ /tmp/dist/otelcol-custom --config=config.yaml
2023-10-24T20:33:16.872Z        info    service@v0.88.0/telemetry.go:84 Setting up own telemetry...
2023-10-24T20:33:16.873Z        info    service@v0.88.0/telemetry.go:201        Serving Prometheus metrics      {"address": ":8888", "level": "Basic"}
2023-10-24T20:33:16.873Z        info    exporter@v0.88.0/exporter.go:275        Development component. May change in the future.        {"kind": "exporter", "data_type": "metrics", "name": "emptyexporter"}
2023-10-24T20:33:16.873Z        info    exporter@v0.88.0/exporter.go:275        Development component. May change in the future.        {"kind": "exporter", "data_type": "logs", "name": "emptyexporter"}
2023-10-24T20:33:16.873Z        info    exporter@v0.88.0/exporter.go:275        Development component. May change in the future.        {"kind": "exporter", "data_type": "logs", "name": "debug"}
2023-10-24T20:33:16.874Z        info    exporter@v0.88.0/exporter.go:275        Development component. May change in the future.        {"kind": "exporter", "data_type": "traces", "name": "emptyexporter"}
2023-10-24T20:33:16.874Z        info    exporter@v0.88.0/exporter.go:275        Development component. May change in the future.        {"kind": "exporter", "data_type": "traces", "name": "debug"}
2023-10-24T20:33:16.874Z        info    exporter@v0.88.0/exporter.go:275        Development component. May change in the future.        {"kind": "exporter", "data_type": "metrics", "name": "debug"}
2023-10-24T20:33:16.877Z        info    service@v0.88.0/service.go:143  Starting otelcol-custom...      {"Version": "1.0.0", "NumCPU": 20}
```

Pay special attention to the `exporter@v0.88.0/exporter.go:*` lines. These lines indicate that the collector has registered our exporter for traces, metrics, and logs. We are now ready to attach a debugger to our custom collector and play around with the received metrics, traces, and logs.

## Debugging the Exporter

The primary reason for constructing a custom version of the collector is to facilitate debugging of your exporter. Now that we have a setup ready for debugging, let's dig into how it can be achieved.

The initial step is to include `debug_compilation: true` in the `dist` section of the `otelcol-builder.yaml` file. This action ensures that the debug symbols are incorporated into the binary during the collector's build, paving the way for debugging the exporter.

```yaml
dist:
  debug_compilation: true
```

After some exploration, I discovered that the [delve](https://github.com/go-delve/delve) tool can be used to debug Go applications. Given the plethora of resources available on using delve for Go debugging, I opted for this method.

You can kickstart the custom collector in debug mode with the following command:

```bash
dlv --listen=:2345 --headless=true --api-version=2 --accept-multiclient --log exec /tmp/dist/otelcol-custom -- --config=config.yaml
```

You should see the following output:

```bash
vscode ➜ /workspaces/first-opentelemetry-exporter $ dlv --listen=:2345 --headless=true --api-version=2 --accept-multiclient --log exec /tmp/dist/otelcol-custom -- --config=config.yaml
API server listening at: [::]:2345
2023-10-24T20:37:18Z warning layer=rpc Listening for remote connections (connections are not authenticated nor encrypted)
2023-10-24T20:37:18Z info layer=debugger launching process with args: [/tmp/dist/otelcol-custom --config=config.yaml]
2023-10-24T20:37:18Z debug layer=debugger Adding target 4264 "/tmp/dist/otelcol-custom --config=config.yaml"
```

> **Note**: An unusual quirk I encountered during my debugging journey was with delve's termination process. Once delve is active, a simple `ctrl+c` doesn't suffice to exit. The workaround I employed was to initiate another terminal and run the command `killall dlv`. If anyone is aware of a more elegant solution to this, I'm eager to hear it!

Subsequently, a `launch.json` file needs to be created within the `.vscode` directory. This file instructs Visual Studio Code on how to connect to `delve` and debug the exporter.

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

To wrap things up, navigate to the `Run and Debug` tab in Visual Studio Code. From the dropdown menu, select `Connect to server` and hit the play button. Visual Studio Code will now establish a connection with the custom collector. This enables you to set breakpoints within your exporter, specifically on the `pushLogs`, `pushMetrics`, or `pushTraces` functions, and observe incoming data in real-time.

## Conclusion

By now, you should have a functional custom collector accompanied by a debuggable exporter. While the current setup offers a basic framework for an exporter, it establishes a robust foundation from which you can develop and expand. Building and debugging your own OpenTelemetry collector exporter not only deepens your understanding of the OpenTelemetry ecosystem but also empowers you to tailor monitoring solutions to your unique needs.

For those seeking inspiration, dig into the vast array of receivers, exporters, and processors available in the wild, courtesy of the [OpenTelemetry Collector](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter) and [OpenTelemetry Collector Contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter).

For a glimpse into the kind of data your exporter might handle, here's an [example JSON](https://opentelemetry.io/docs/specs/otel/protocol/file-exporter/#examples).

[^1]: changing the `output_path`

    While setting up, I found it inconvenient that the `output_path` was directed to a location within the     container. In an attempt to redirect it to a local machine location, I set it as `/workspaces/opentelemetry-embedding-exporter/otelcol-custom`. Unfortunately, this adjustment was met with an error     upon executing the build command:
    
    ```bash
    Error: failed to compile the OpenTelemetry Collector distribution: exit status 1. Output:
    error obtaining VCS status: exit status 128
        Use -buildvcs=false to disable VCS stamping.
    ```
    
    Although I believe there should be a way to make this work, I chose not to go down a troubleshooting     rabbit hole. If anyone has insights or solutions regarding this issue, I'm open to suggestions.
