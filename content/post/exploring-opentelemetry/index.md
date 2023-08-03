---
title: "Exploring Observability with OpenTelemetry: A Comprehensive Guide"
description: "Explore OpenTelemetry with a deep dive into setup, configuration, service integration, and local telemetry"
slug: exploring-opentelemetry
date: 2023-08-02 00:00:00+0000
original: true
image: cover.jpg
categories:
    - Development
tags:
    - OpenTelemetry
    - .NET
---

[OpenTelemetry](https://opentelemetry.io/) is an open-source project under the umbrella of the Cloud Native Computing Foundation (CNCF). It was designed with the intent to create a comprehensive and standardized framework for observability in cloud-native software. In today's world, as modern applications are moving towards distributed and microservices architectures, understanding the interactions and performance of these services has become notably challenging. This is precisely the issue that OpenTelemetry strives to address.

OpenTelemetry offers a unified method to collect, process, and export telemetry data, including metrics, logs, and traces, for further analysis. By providing standard, vendor-neutral APIs, developers are able to instrument their code just once, and subsequently send the data to any backend supporting the OpenTelemetry protocol.

I've found OpenTelemetry to be a highly effective tool for observability and have been an advocate for its use in the projects I contribute to for quite some time. Its popularity, however, does make navigating its usage slightly less than straightforward. Hence, in this article, I intend to share my experiences with OpenTelemetry, walking you through the setup process, configurations, and service integrations. Moreover, I will guide you on utilizing OpenTelemetry locally, providing insights into the data it generates without needing internet connectivity. You can find all the code examples used in this article in my [GitHub repository](https://github.com/droosma/exploring-opentelemetry).

While this article delves into various facets of OpenTelemetry, it doesn't cover the specific implementation of telemetry in applications—there are plenty of online resources that delve into that. However, you can look forward to a subsequent blog post where I will discuss my approach to integrating telemetry into my applications.

## Producer

To experiment with OpenTelemetry, an application that generates data is required. For the simplest OpenTelemetry setup, start by creating a new directory and executing the following commands:

```powershell
dotnet new web
dotnet add package OpenTelemetry.Exporter.Console
dotnet add package OpenTelemetry.Extensions.Hosting

```

Next, replace the code from `Program.cs` with the following:

```csharp
using OpenTelemetry.Logs;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);
var resourceBuilder = ResourceBuilder.CreateDefault()
                                     .AddService("producer", "v1.0.0");

builder.Services.AddLogging(x => x.AddOpenTelemetry(options => options.SetResourceBuilder(resourceBuilder)
                                                                      .AddConsoleExporter()
                                                   ));
builder.Services
       .AddOpenTelemetry()
       .WithTracing(x => x.AddConsoleExporter()
                          .AddSource("producer")
                          .SetResourceBuilder(resourceBuilder))
       .WithMetrics(x => x.AddConsoleExporter()
                          .AddMeter("producer")
                          .SetResourceBuilder(resourceBuilder));

var app = builder.Build();

app.Run();
```

In this code snippet, we're configuring a .NET application with OpenTelemetry for observability. We start off with establishing a ResourceBuilder to define our application metadata—specifically its name and version.

We then inject OpenTelemetry into our logging services, associating it with our defined ResourceBuilder and specifying a console exporter to direct our logs to the console.

Taking things further, we integrate OpenTelemetry into our services, enriching them with both tracing and metrics capabilities. For tracing, we identify the source of the trace data as "producer" and, similar to logging, direct the output to the console. For metrics, we set up a meter labeled "producer" and channel the metrics data to our console too.

When you kick off your application using `dotnet run`, you should see something like this pop up in your console:

```text
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: https://localhost:5001
LogRecord.Timestamp:               2023-08-02T16:07:25.1377163Z
LogRecord.CategoryName:            Microsoft.Hosting.Lifetime
LogRecord.LogLevel:                Information
LogRecord.Body:                    Now listening on: {address}
LogRecord.Attributes (Key:Value):
    address: https://localhost:5001
    OriginalFormat (a.k.a Body): Now listening on: {address}
LogRecord.EventId:                 14
LogRecord.EventName:               ListeningOnAddress

Resource associated with LogRecord:
service.name: producer
service.namespace: v1.0.0
service.instance.id: b925339d-36d3-4a0c-abcf-aec39b029631
telemetry.sdk.name: opentelemetry
telemetry.sdk.language: dotnet
telemetry.sdk.version: 1.5.1
```

Now that you've handled the basics, let's advance to a more useful and maintainable setup.

## OpenTelemetry Collector

In the previous example, we utilized the [OpenTelemetry.Exporter.Console](https://www.nuget.org/packages/OpenTelemetry.Exporter.Console) package to display telemetry. While it's suitable for initial local development with OpenTelemetry, as you expand your application's observability, the volume of data logged to the console can become excessive and unreadable.

Searching for [`OpenTelemetry.Exporter` on nuget.org](https://www.nuget.org/packages?q=OpenTelemetry.Exporter) yields a variety of exporters for your telemetry output. However, simply adding these packages doesn't strike me as the optimal solution. Except for occasional side projects, most software I develop gets deployed across multiple environments with diverse requirements. For instance, during development, I might prefer console output for telemetry. But in test or acceptance stages, there might be a central Application Performance Monitoring (APM) service that saves all data but only retains it for a week. In a production environment, we might even need to send distinct telemetry to various APM SaaS providers.

Using `OpenTelemetry.Exporter.*` packages would necessitate substantial conditional coding in your application to cater to these varied environments. My position is that an application shouldn't be concerned with the destination of it's telemetry—it should simply produce it. This is where the [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) comes in handy, taking on the responsibility of exporting your telemetry data. As an added advantage, if you're exporting to multiple APM services, the Collector can offload this task from your application, thereby increasing efficiency.

We'll replace the Console Exporter with [OpenTelemetry.Exporter.OpenTelemetryProtocol](https://www.nuget.org/packages/OpenTelemetry.Exporter.OpenTelemetryProtocol), and `AddConsoleExporter()` with `AddOtlpExporter()`. The OTLP exporter defaults to `http://localhost:4317` for telemetry output using the GRPC protocol, but we can configure it for any desired destination as follows:

```csharp
builder.AddOtlpExporter(exporterOptions => exporterOptions.Endpoint = new Uri("172.17.0.10:1234"));
```

For the subsequent steps, I suggest using the Producer from my repository, as it's already correctly configured.

Our current configuration is essentially logging into the void, as there's nothing listening at `http://localhost:4317`. Let's address this by launching the OpenTelemetry Collector and setting up the appropriate port forwarding on localhost to the container, thereby allowing it to receive the telemetry data.

Running the OpenTelemetry Collector in a Docker container is my preference, which involves using the following `docker-compose.yaml`:

```docker-compose
version: '3'
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib
    volumes:
      - ./otel-collector-config.yaml:/etc/otelcol-contrib/config.yaml
    ports:
      - 4317:4317
```

The Docker Compose file mounts a configuration file for the container. This file configures the exporters and receivers for the OpenTelemetry Collector. For starters, we'll use this basic configuration:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
processors:
  batch:
exporters:
  logging:
    verbosity: detailed
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [logging]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [logging]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [logging]
```

This setup includes an OTLP receiver using the GRPC protocol, a batch processor, and a logging exporter. We then establish a pipeline for each telemetry type or [signal](https://opentelemetry.io/docs/concepts/signals/). Each pipeline contains receivers (IN), processors (TRANSFORM), and exporters (OUT). We're using the same single receiver, processor, and exporter for all three pipelines.

Running the Docker Compose file and the application `dotnet run --project .\producer\Producer.csproj` should yield a familiar console output:

```text
exploring-opentelemetry-otel-collector-1  | LogRecord #0
exploring-opentelemetry-otel-collector-1  | ObservedTimestamp: 2023-08-02 17:10:57.6799786 +0000 UTC
exploring-opentelemetry-otel-collector-1  | Timestamp: 2023-08-02 17:10:57.6799786 +0000 UTC
exploring-opentelemetry-otel-collector-1  | SeverityText: Information
exploring-opentelemetry-otel-collector-1  | SeverityNumber: Info(9)
exploring-opentelemetry-otel-collector-1  | Body: Str(Now listening on: {address})
exploring-opentelemetry-otel-collector-1  | Attributes:
exploring-opentelemetry-otel-collector-1  |      -> dotnet.ilogger.category: Str(Microsoft.Hosting.Lifetime)
exploring-opentelemetry-otel-collector-1  |      -> Id: Int(14)
exploring-opentelemetry-otel-collector-1  |      -> Name: Str(ListeningOnAddress)
exploring-opentelemetry-otel-collector-1  |      -> address: Str(https://localhost:5001)
exploring-opentelemetry-otel-collector-1  | Trace ID:
exploring-opentelemetry-otel-collector-1  | Span ID:
exploring-opentelemetry-otel-collector-1  | Flags: 0
```

This situation seems unchanged, except we've introduced more dependencies. However, the OpenTelemetry Collector's strength begins to show. By adding more exporters to the configuration file, we keep the application code unchanged.

In the Docker Compose file, you may have spotted the `-contrib`` suffix in the Docker Compose file. This originates from the structure of the OpenTelemetry Collector project, a component of the OpenTelemetry project. This larger project actively encourages community contributions, including their own exporters, processors, and receivers, which are consolidated in the [OpenTelemetry Collector Contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib) project. This community-driven structure makes expanding your telemetry exporting capabilities as straightforward as adding your desired receiver, processor, or exporter into your configuration file. For an extensive list of supported [receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver), [processors](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor), and [exporters](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter), you can refer to the respective links.

## Working locally

Despite my admiration for products like Honeycomb and [DataDog](https://www.datadoghq.com/), I prefer not to be dependent on them for local development. Given that internet blackouts can be frequent during my commute, I find it important to be able to work offline. In these circumstances, the OpenTelemetry Collector proves highly beneficial.

There exist numerous impressive Docker images that can be configured for use with the collector.

### Tracing

For tracing, the solution is relatively straightforward. There are many open source systems that allow you to run an instance locally. I frequently use [Jaeger](https://www.jaegertracing.io/), an excellent open source tracing system. There is a [docker image](https://hub.docker.com/r/jaegertracing/all-in-one) that allows us to spin up a local instance of Jaeger. Let's adjust our `docker-compose.yaml` file to utilize this image by adding the following:

```yaml
  otel-collector:
    ....
    links:
      - jaeger
  jaeger:
    image: jaegertracing/all-in-one
    ports:
      - 14250
      - 16686
```

Next, let's modify the configuration file to use the Jaeger exporter:

```yaml
exporters:
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true
service:
  pipelines:
    traces:
      ...
      exporters: [logging, jaeger]
```

After restarting your docker-compose and running the application, navigate to [http://localhost:5000/trace](http://localhost:5000/trace). The Jaeger UI should be available at [http://localhost:16686](http://localhost:16686), where you should see the traces of the producer appearing in the UI.

### Metrics

Metrics are a bit more complex to handle than tracing. Although there are several open source systems available, they tend not to be as straightforward to set up as Jaeger. For local development, I've found that the best solution is to use [Prometheus](https://prometheus.io/). But, as Prometheus operates on a pull-based system, the configuration is slightly different.

In the `docker-compose.yml` we add the following:

```yaml
  prometheus:
    image: prom/prometheus
    ports:
      - 9090:9090
    volumes:
      - ./prometheus-config.yaml:/etc/prometheus/prometheus.yml
    links:
      - otel-collector
```

It's crucial to note that we're adding a `links` component to the Prometheus container. This is because we need to be able to reach the OpenTelemetry Collector from the Prometheus container.

In the `otel-collector-config.yaml` we configure the collector to use Prometheus as an exporter:

```yaml
exporters:
  prometheus:
    endpoint: 0.0.0.0:9123
service:
  pipelines:
    metrics:
      ...
      exporters: [logging, prometheus]
```

Here, the endpoint doesn't signify the endpoint of the Prometheus instance, but rather an endpoint of the OpenTelemetry Collector. The Prometheus exporter will initiate a server on this endpoint, which Prometheus can then scrape.

Lastly, we need to configure Prometheus to scrape the OpenTelemetry Collector. This is accomplished in the `prometheus-config.yaml` file:

```yaml
scrape_configs:
  - job_name: 'otel-collector'
    scrape_interval: 5s
    static_configs:
      - targets: ['otel-collector:9123']
```

As with the Jaeger exporter, we need to restart docker-compose and run the application again. This time, visit [http://localhost:5000/metric](http://localhost:5000/metric). The Prometheus UI should be accessible at [http://localhost:9090](http://localhost:9090). If you enter `random_number_count` into the search field, you should see the metrics appear in the UI.

### Logging

The logging signal specification was among the last to stabilize in the OpenTelemetry project, happening on the [11th of may 2022](https://github.com/open-telemetry/opentelemetry-proto/commit/d6481eac0579c32ba4f58ee4de9ce0dd29021462). During most of my explorations with OpenTelemetry, many systems implementing the specification were yet to support this feature. I've only utilized the logging signal in a production environment once, and during that period, didn't spend much time operating a local logging system. However, while preparing this article, I managed to successfully use [Loki](https://grafana.com/oss/loki/) as an export target for logging.

Like the other signals, we need to include the Loki docker image in the `docker-compose.yml` file:

```yaml
otel-collector:
  ...
  links:
    ...
    - loki
loki:
  image: grafana/loki:v1.3.0
  volumes:
    - ./loki.yaml:/etc/config/loki.yaml
  entrypoint:
    - /usr/bin/loki
    - -config.file=/etc/config/loki.yaml
  ports:
    - 3100:3100
```

Ensure that Loki is linked to the OpenTelemetry Collector, so that the collector can access Loki. Loki also needs a configuration file. I found the following [config online](https://blog.ruanbekker.com/blog/2020/08/13/getting-started-on-logging-with-loki-using-docker/). This might contain more than required, but it works. As I'm not entirely familiar with Loki, I won't go into detail about the configuration. You can find it in the source article, or in my [repository](https://github.com/droosma/exploring-opentelemetry/loki.yaml).

Since Loki is headless, if you want to visualize your logs, you'll need a UI. [Grafana](https://grafana.com/) can serve this purpose. We add the following to the `docker-compose.yml` file:

```yaml
  grafana:
    image: grafana/grafana:7.1.1
    volumes:
    - ./grafana-datasources.yaml:/etc/grafana/provisioning/datasources/datasource.yml
    ports:
    - 3000:3000
    links:
      - loki
```

Here, we link Grafana to Loki as it must be able to access it. We also have to configure Grafana to use Loki as a data source. This is done in the `grafana-datasources.yaml` file:

```yaml
# config file version
apiVersion: 1

deleteDatasources:
  - name: loki

datasources:
- name: loki
  type: loki
  access: proxy
  orgId: 1
  url: http://loki:3100
  basicAuth: false
  isDefault: true
  version: 1
  editable: false
```

The final configuration step is to set the OpenTelemetry Collector to use Loki as an exporter. This is done in the `otel-collector-config.yaml` file:

```yaml
exporters:
  loki:
    endpoint: http://loki:3100/loki/api/v1/push
```

And with that, everything should be ready. We just need to restart docker-compose and run the application again, this time visiting [http://localhost:5000/Log](http://localhost:5000/Log). Grafana should be available at [http://localhost:3000/explore](http://localhost:3000/explore). If you enter `{job="producer"}` into the query field, you should see the logs appear.

As previously stated, my experience with Loki is limited. Therefore, if you detect any inaccuracies or oversights, feel free to reach out to me directly.

### Application Performance Monitoring

We now have all the individual signals functioning locally, a feat in itself, but let's aim higher. In a perfect world, we would want to emulate a full APM solution akin to Honeycomb or DataDog. Regrettably, after an extensive search, I'm yet to stumble upon a solution that fits the bill.

Take [Uptrace](https://uptrace.dev/) for instance, a service I managed to get up and running. If you follow the steps I used and run the `docker-compose` available [here](https://github.com/droosma/exploring-opentelemetry/uptrace/docker-compose.yml), you should see your telemetry data appear in the UI as well. However, the interface leaves much to be desired, being somewhat unintuitive. In fact, I found myself investing more time in wrestling with the UI to get my telemetry data than if I had just switched between individual services.

Another option I've come across is Grafana Labs. This company, which offers multiple products, three of which I've mentioned in this post, might present a viable solution. Hypothetically, Jaeger could be substituted with Grafana Tempo, one of their offerings. However, I must admit I haven't tested this out yet. The endgame, it seems, would involve integrating all of Grafana's products and configuring a personalized APM solution via the Grafana UI. This approach could be worthwhile if you're already utilizing Grafana products for your production systems, as it might enable you to replicate your production APM environment locally. But be prepared - this endeavor would require a substantial investment of time and effort, commodities I've not yet been able to spare.

Lastly, I've identified a few more APM solutions that I haven't explored as yet:

- SigNoz, documentation for which can be found [here](https://signoz.io/docs/install/docker/)
- Elastic APM, the quick start guide is available [here](https://www.elastic.co/guide/en/apm/get-started/current/quick-start-overview.html)

To sum it up, finding the perfect local APM solution remains a challenging goal. However, with continued investigation and experimentation, I believe we'll get there.
