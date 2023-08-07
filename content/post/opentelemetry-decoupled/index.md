---
title: "OpenTelemetry Refined: The Collector’s Approach to Decoupled Data Exporting"
description: "Stop referencing your OpenTelemetry exporters in your applications. In this article, we'll explore a great way to decouple them using the OpenTelemetry Collector"
slug: opentelemetry-decoupled
date: 2023-08-07 00:00:00+0000
original: true
image: cover.jpg
categories:
    - Development
tags:
    - OpenTelemetry
    - .NET
---

As the tech landscape progressively adopts OpenTelemetry as the go-to standard for application telemetry, I feel compelled to bring attention to an all too common issue I encounter in various blog posts and documentation - the neglect of the OpenTelemetry Protocol as a singular exporter. When you peruse the official OpenTelemetry documentation or even delve into an article about [exporters](https://opentelemetry.io/docs/instrumentation/net/exporters/), you often stumble upon a direct configuration of OpenTelemetry tracing with a [Zipkin](https://zipkin.io/) exporter.

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenTelemetry()
    .WithTracing(b =>
    {
        b.AddZipkinExporter(o =>
        {
            o.Endpoint = new Uri("your-zipkin-uri-here");
        })
        // The rest of your setup code goes here too
    });
```

A quick search for [`OpenTelemetry.Exporter` on nuget.org](https://www.nuget.org/packages?q=OpenTelemetry.Exporter) brings forth an array of exporters suitable for your telemetry output. However, simply incorporating these packages hardly seems like the most efficient approach. The software I develop is generally deployed across multiple environments, each bearing unique requirements. For example, during development, console output might be my telemetry method of choice. But when it comes to testing or acceptance stages, we could be looking at a centralized Application Performance Monitoring (APM) service that holds onto data for a limited period of a week. The story changes further in production, where we may need to dispatch discrete telemetry to multiple APM SaaS providers.

If you choose to use `OpenTelemetry.Exporter.*` packages, your application will demand a fair share of conditional coding to meet these diverse environmental needs. I firmly believe that applications should focus on producing telemetry rather than worry about its destination. This is where the [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) takes center stage, relieving your application of the exporting task. A bonus point to consider is the Collector's ability to handle multiple APM services, thus enhancing your application's overall efficiency.

## OpenTelemetry Collector

Rather than directly employing the aforementioned exporters in our application and configuration, we shift to using [OpenTelemetry.Exporter.OpenTelemetryProtocol](https://www.nuget.org/packages/OpenTelemetry.Exporter.OpenTelemetryProtocol) as our exporter of choice. This necessitates modifying `AddZipkinExporter()` to `AddOtlpExporter()` in the configuration. The default configuration of the OTLP exporter sends all telemetry to `http://localhost:4317` via gRPC, but it can be tailored for any destination:

```csharp
.AddOtlpExporter(exporterOptions => exporterOptions.Endpoint = new Uri("172.17.0.10:1234"));
```

If you've implemented these changes, give yourself a pat on the back! Your application now logs into oblivion since nothing is listening on `http://localhost:4317`. Let's rectify this by launching the OpenTelemetry Collector and setting up suitable port forwarding on the container, enabling it to receive the telemetry data.

I prefer running the OpenTelemetry Collector in a Docker container, which can be done using the following `docker-compose.yaml`:

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

In the Docker Compose file, we mount a configuration file for the container. This file orchestrates the exporters, processors and receivers for the OpenTelemetry Collector. Assuming that you're utilizing Zipkin as demonstrated earlier, we can resort to this simple configuration:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
processors:
  batch:
exporters:
  zipkin:
    endpoint: your-zipkin-uri-here
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [zipkin]
```

Here, we've set up an OTLP receiver using the GRPC protocol, a batch processor, and the Zipkin exporter. Next, we set up a pipeline for the traces [signal](https://opentelemetry.io/docs/concepts/signals/), which includes receivers (IN), processors (TRANSFORM), and exporters (OUT). In this instance, we've only configured the traces pipeline. For a more comprehensive configuration overview, click [here](https://opentelemetry.io/docs/collector/configuration/). Once everything is in place and the Docker Compose file is running, your traces should be visible in Zipkin as before.

You might be thinking, the situation seems unchanged, apart from the introduction of additional dependencies and configuration. But, the power of the OpenTelemetry Collector is about to shine. By integrating more exporters into the configuration file, the application code remains unaltered.

You might have noticed the `-contrib` suffix in the Docker Compose file. This stems from the structure of the OpenTelemetry Collector project, a part of the larger OpenTelemetry project. This project encourages community contributions, including the creation of their own exporters, processors, and receivers, which are grouped together in the [OpenTelemetry Collector Contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib) project. This community-oriented structure simplifies the expansion of your telemetry exporting capabilities. All you need to do is include your preferred receiver, processor, or exporter in your configuration file. For a comprehensive list of supported [receivers](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver), [processors](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor), and [exporters](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter), refer to the respective links.

## Working locally

Despite my admiration for products like [Honeycomb](https://www.honeycomb.io/) and [DataDog](https://www.datadoghq.com/), I prefer not to be dependent on them for local development. Given that internet blackouts have been frequent during my past commutes, I find the ability to work offline very important. In these circumstances, the OpenTelemetry Collector proves highly beneficial.

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
      - 14250:14250
      - 16686:16686
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
