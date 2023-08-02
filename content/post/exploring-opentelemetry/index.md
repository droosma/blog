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

In this article I will be describing how I use OpenTelemetry as my go-to tool for observability. I will be going into detail about how to set it up, how to configure it, and how to integrate it with some of the services I use. I will also be showing you how to use it locally to get a feel for the data it produces. You can find all the code I use in this article in my [GitHub repository](https://github.com/droosma/exploring-opentelemetry)

I won't be going into how to use OpenTelemetry in your application, there are lots of resources online, do keep you eyes out for a future blog post about my particular way of doing this

## Producer

To play around with OpenTelemetry is nice to have a application that can produce some data, here is the absolute minimum you need to do to get OpenTelemetry up and running

The NuGet package references:

```xml
<ItemGroup>
    <PackageReference Include="OpenTelemetry" Version="1.5.1" />
    <PackageReference Include="OpenTelemetry.Exporter.Console" Version="1.5.1" />
    <PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.5.1" />
</ItemGroup>
```

And have the following code in your `Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);
var resourceBuilder = ResourceBuilder.CreateDefault()
                                     .AddService("producer", "v1.0.0");

builder.Services.AddLogging(x => x.AddOpenTelemetry(options => options.SetResourceBuilder(resourceBuilder)
                                                                      .AddConsoleExporter()
                                                   ));
builder.Services.AddOpenTelemetry()
       .WithTracing(x => x.AddConsoleExporter()
                          .AddSource("producer")
                          .SetResourceBuilder(resourceBuilder))
       .WithMetrics(x => x.AddConsoleExporter()
                          .AddMeter("producer")
                          .SetResourceBuilder(resourceBuilder));

var app = builder.Build();

app.Run();
```

When you run this code, you should see the following output in the console:

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

Great, now lets quickly move on to something a bit more usable and maintainable.

## OpenTelemetry Collector

In the sample above we used the [console exporter](https://www.nuget.org/packages/OpenTelemetry.Exporter.Console) to output the telemetry. This is great for local development when you are just starting out with OpenTelemetry, but as soon as you start implementing more observability in your application the information that is being logged to the console will become overwhelming.

If you search for [`OpenTelemetry.Exporter` on nuget.org](https://www.nuget.org/packages?q=OpenTelemetry.Exporter) you will see a long list of exporters that you can use to output your telemetry to. The way I see it, adding these packages is not the way to go. Except the occasional side project, most of the software I write is deployed on multiple environments with varying requirements. For example, during development I might want to output the telemetry to the console, but in test or acceptance we might have a central Application Performance Monitoring (APM) service that is configured to store everything but only keep it for 7 days. And in production we might even have to send the telemetry to multiple APM Saas providers.

If you would use the `OpenTelemetry.Exporter` packages you would have to add a lot of conditional code to your application to support all these different environments.
The way I see it, the application should not be concerned with where the telemetry is going, it should just produce it. This is where the [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) comes in.

Instead of the Console Exporter we will be using [OpenTelemetry.Exporter.OpenTelemetryProtocol](https://www.nuget.org/packages/OpenTelemetry.Exporter.OpenTelemetryProtocol)
and instead of `AddConsoleExporter()` we will be using `AddOtlpExporter()`.
By default the OTLP exporter will output the telemetry to `http://localhost:4317` using the GRPC protocol, but we can configure it to output to any destination we want like so:

```csharp
builder.AddOtlpExporter(exporterOptions => exporterOptions.Endpoint = new Uri("172.17.0.10:1234"));
```

Ok, so now we have it logging to the void, as we don't have anything listening on `http://localhost:4317`. Lets fix that by running the OpenTelemetry Collector.

I like to run the OpenTelemetry Collector in a Docker container, so I have a `docker-compose.yaml` file contains the following:

```docker-compose
otel-collector:
  image: otel/opentelemetry-collector-contrib
  volumes:
    - ./otel-collector-config.yaml:/etc/otelcol-contrib/config.yaml
  ports:
    - 4317
```

As you can see the docker-compose file mounts a configuration file to the container. This file will be used by the OpenTelemetry Collector to configure the exporters and receivers.
Let's start with a simple configuration:

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

We configure an OTLP receiver that uses the GRPC protocol the default for `AddOtlpExporter()`, a batch processor and a logging exporter. Next we configure a pipeline for each type of telemetry or [signal](https://opentelemetry.io/docs/concepts/signals/). Each pipeline has a receivers (IN), processors (TRANSFORM) and exporters (OUT). In this case we are using the same single receiver, processor and exporter for all three pipelines.

If you now run you docker-compose file and run the application with: `dotnet run --project .\producer\Producer.csproj`, you should see a very familiar output in the console:

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

Great, we have the same as before, but now with more dependencies. But now we start to see the power of the OpenTelemetry Collector. We can now add more exporters to the configuration file and the application will not have to change at all.

The keen eyed will have noticed that the image used in the docker-compose has a `-contrib` suffix. This is because the OpenTelemetry Collector is a project that is maintained by the OpenTelemetry project, but the community is free to add their own exporters and receivers. These are maintained in the [OpenTelemetry Collector Contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib) project. This project contains a lot of exporters and receivers for a lot of different systems and products.

Now exporting you telemetry is a simple as adding a new exporter to the configuration file, let's try adding a [Honeycomb](https://www.honeycomb.io/) exporter:

```yaml
exporters:
  otlp/honeycomb:
    endpoint: "api.honeycomb.io:443"
    headers:
      "x-honeycomb-team": "YOUR_API_KEY"
```

And changing the pipeline to use the new exporter:

```yaml
service:
  pipelines:
    ....
    logs:
      ....
      exporters: [logging, otlp/honeycomb]
```

For a list of supported exporters see [here](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter)

## Working locally

While I love products like Honeycomb or [DataDog](https://www.datadoghq.com/), I don't want to have to rely on them for my local development. Having frequent internet blackouts during my commute, I want to be able to work offline. Here too the OpenTelemetry Collector comes to the rescue.

There are some awesome docker images out there that we can configure the collector to use.

### Tracing

For tracing it relatively simple, there a lot open source systems that allow you to run an instance locally. I often use [Jaeger](https://www.jaegertracing.io/) this is a great open source tracing system. There is a [docker image](https://hub.docker.com/r/jaegertracing/all-in-one) that we can use to spin-up a local instance of Jaeger. Let change our docker-compose file to use this image by adding the following:

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

And change the configuration file to use the Jaeger exporter:

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

Restarting your docker-compose file, running the application and visit [http://localhost:5000/trace](http://localhost:5000/trace). Jaeger UI should be available at [http://localhost:16686](http://localhost:16686) there you should see the traces of the producer appear in the UI.

### Metrics

For metrics it is a little bit more difficult, there are some open source systems out there, but they are not as easy to use as Jaeger. I have found that the best way to work locally is to use the [Prometheus](https://prometheus.io/).
But as Prometheus is a pull based system, the configuration is a bit different.

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

Pay special attention to the fact that we add a `links` component to the Prometheus container. This is because we want to be able to reach the OpenTelemetry Collector from the Prometheus container. 

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

Again the endpoint here does not signify the endpoint of the Prometheus instance, but a endpoint of the OpenTelemetry Collector. The Prometheus exporter will start a server on this endpoint that Prometheus can scrape.

Finally we need to configure Prometheus to scrape the OpenTelemetry Collector. This is done in the `prometheus-config.yaml` file:

```yaml
scrape_configs:
  - job_name: 'otel-collector'
    scrape_interval: 5s
    static_configs:
      - targets: ['otel-collector:9123']
```

And just like with Jaeger exporter, we just need to restart docker-compose and run the application again, this time visit [http://localhost:5000/metric](http://localhost:5000/metric). Prometheus UI should be available at [http://localhost:16686](http://localhost:16686) where if you enter `random_number_count` into the search field you should see the metrics appear in the UI.

### Logging

As the logging signal is not that old yet, there are not that many open source systems out there that support it.

But I have gotten it to work with [Loki](https://grafana.com/oss/loki/), but don't have much experience with it, so if you see something wrong, please let me know.

First we need to add the Loki docker image to the `docker-compose.yml` file:

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

We make sure to link loki to the OpenTelemetry Collector, so that the collector can reach Loki. And as you might have spotted Loki also needs a configuration file, I have found the following [config online](https://blog.ruanbekker.com/blog/2020/08/13/getting-started-on-logging-with-loki-using-docker/). So this might contain more than we need, but it works. As I'm not really familiar with Loki, I'm not going to go into detail about the configuration, you can find it in the source article, or in my [repository](https://github.com/droosma/exploring-opentelemetry/loki.yaml).

As Loki is headless, if you want to actually see your logs we need a UI. For this we can use [Grafana](https://grafana.com/). We add the following to the `docker-compose.yml` file:

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

Here we link Grafana to Loki as it needs to be able to reach it. We also need to configure Grafana to use Loki as a datasource. This is done in the `grafana-datasources.yaml` file:

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

The last config we need to do is to configure the OpenTelemetry Collector to use Loki as an exporter. This is done in the `otel-collector-config.yaml` file:

```yaml
exporters:
  loki:
    endpoint: http://loki:3100/loki/api/v1/push
```

And just like that, everything should be good to go. We just need to restart docker-compose and run the application again, this time visit [http://localhost:5000/Log](http://localhost:5000/Log). Grafana be available at [http://localhost:3000/explore](http://localhost:3000/explore) if you where if you enter `{job="producer"}` into the query field you should see the logs appear.

### Local APM

Alright now we have all the individual signals working locally awesome, but ideally we would emulate a full APM like Honeycomb or DataDog. I have been searching for this for quite some time, and to be honest, I have not found a good solution that appeals to me yet.

#### Uptrace

[Uptrace](https://uptrace.dev/)
I have been able to get this running, and you should too if you run the `docker-compose` [here](https://github.com/droosma/exploring-opentelemetry/uptrace/docker-compose.yml). I see my telemetry appear in the UI, but I find the interface unintuitive, I end up spending more time trying to figure out how to get my telemetry from the UI than I would if I just switch between the individual services as mentioned above.

#### Grafana Labs

As Grafana labs already has multiple products, 3 of which I use in this post. Jaeger should be replaceable by [Grafana Tempo](https://grafana.com/oss/tempo/) but I have not tried it yet.
In the end I think you would still need to combine all of Grafana's products and configure your own APM solution thought the Grafana UI. This might very well be a viable option if you are already using Grafana products for you production systems, as it might be worth while get to a state where your production APM is reproducible locally. But this would require a substation amount of time. I have not been in a situation where I had the time to do this.

#### Some others

These are some other APM solutions I have found, but have not tried yet:

- [SigNoz](https://signoz.io/docs/install/docker/)
- [Elastic APM](https://www.elastic.co/guide/en/apm/get-started/current/quick-start-overview.html)