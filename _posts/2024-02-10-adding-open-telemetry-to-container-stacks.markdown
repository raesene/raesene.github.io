---
layout: post
title: "Adding Open Telemetry to Container Stacks" 
date: 2024-02-10 10:27:00 +0100
comments: false
categories: OpenTelemetry
---

This year, I've started looking at how [observability can work well for security](https://www.linkedin.com/pulse/security-observability-match-made-heaven-rory-mccune-mej4e%3FtrackingId=MrZn6Pw%252FpmXQjCGKT1hxKA%253D%253D/?trackingId=MrZn6Pw%2FpmXQjCGKT1hxKA%3D%3D) and as part of that I've been investigating Open Telemetry, to understand more about how it works. 

So when I noticed in recent Kubernetes release notes that Open Telemetry support was being added, I decided to take a look at how it's being integrated in k8s and other parts of container stacks.

This blog is just some notes about how to get it set up, and some of the things I've noticed along the way.

## Basic Architecture

There's essentially 3 elements to the architecture of a basic observability stack. We've got sources of Telemetry (e.g. logs, metrics, traces) which in this case will be services like Kubernetes and Docker, a collector to gather and process that telemetry, and then one or more backends to send the information to. 

![Basic OTel Architectuer]({{site.url }}/assets/media/container-otel-architecture.png)

For the sources of telemetry in this case we're going to rely on their OTel integrations, which are built into the software. The [Open Telemetry Collector](https://opentelemetry.io/docs/collector/) is our collector which will receive the data, process it and then forward to our backends. Then for backends to demonstrate having multiple ones setup, I used [Jaeger](https://www.jaegertracing.io/) and [Datadog](https://www.datadoghq.com/) (full disclosure, I work for Datadog :) ).

## Setting up the OTel support in Kubernetes

To test this out in Kubernetes I'm going to make use of [KinD](https://kind.sigs.k8s.io/) to create a local cluster. A relatively recent version of Kubernetes is needed as the OTel support has only been added in the last few releases (alpha in 1.22, beta in 1.27). It's not currently at release level so we need to give the API server a feature flag to enable it. If you want some more background on how tracing is being added to Kubernetes, it's worth reading the [KEP](https://github.com/kubernetes/enhancements/tree/master/keps/sig-instrumentation/647-apiserver-tracing).

This is the KinD configuration I used to create the cluster. In addition to the feature flag, we need a mount to provide the configuration file to the API server.

```YAML
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
featureGates:
  "APIServerTracing": true
nodes:
- role: control-plane
  extraMounts:
  - hostPath: /home/rorym/otel
    containerPath: /otel
    propagation: None
  kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    apiServer:
      extraArgs:
        tracing-config-file: "/otel/config.yaml"
      extraVolumes:
        - name: "otel"
          hostPath: "/otel"
          mountPath: "/otel"
          readOnly: false
          pathType: "Directory"
```

The `tracing-config-file` is the key part here, it's telling the API server where to find the configuration file for the OTel support. The sample file I created looks like this.

```YAML
apiVersion: apiserver.config.k8s.io/v1beta1
kind: TracingConfiguration
endpoint: 192.168.41.107:4317
samplingRatePerMillion: 1000000
```

There's a couple of important settings here. The first one is the `endpoint` which is the address of the OTel collector. The second is the `samplingRatePerMillion` which is the rate at which to sample traces. In this case I'm sampling 100% of traces, but in a real-world scenario you'd want to sample a smaller percentage to avoid overwhelming your backend.

## Setting up the OTel Collector

Next step is to setup the OTel collector to receive the traces from the cluster. We need a configuration file for the collector, which looks like this.

```YAML
receivers:
  otlp: # the OTLP receiver the app is sending traces to
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
processors:
  batch:

exporters:
  otlp/jaeger: # Jaeger supports OTLP directly
    endpoint: 192.168.41.107:44317
    tls:
      insecure: true
  datadog:
    api:
      key: "API_KEY_HERE"

service:
  pipelines:
    traces/dev:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/jaeger, datadog]
```

The `receivers` sections has the ports to listen on , with `4317` and `4318` being defaults. In this case we're deploying the collector on a different host to the cluster, so we'll listen on all interfaces.

Next up we define the exporters for the traces. In this case we're going to forward traces to `jaeger` on a non-standard port (44317) and to `datadog`. The `datadog` exporter needs an API key to be able to send the traces to the backend.

Finally we define a pipeline to process the traces. In this case we're going to process all traces and send them to both `jaeger` and `datadog`.

To run the collector we can then just use this docker command

```bash
docker run --rm --name collector -d -v $(pwd)/config.yaml:/etc/otelcol-contrib/config.yaml -p 4317:4317 -p 4318:4318 -p 55679:55679 otel/opentelemetry-collector-contrib:0.93.0
```

## Setting up Jaeger

For demo purpose we can just run Jaeger using Docker. The command to run it is

```bash
docker run --rm -d -e COLLECTOR_ZIPKIN_HOST_PORT=:9411 -p 16686:16686 -p 44317:4317 -p 44318:4318 -p 49411:9411 jaegertracing/all-in-one:latest
```

As I'm running both containers on the same host, I'm using non-standard ports to avoid conflicts.


## Viewing Traces

Now we've got the cluster up and running and our OTel collector and backends setup, we can start to see traces. This is a screenshot of how the traces look in Datadog.

![Datadog trace list]({{site.url }}/assets/media/datadog-k8s-trace-list.png)

There's quite a bit of information in these traces, showing the internal operations of the cluster. We can see the schedulers and controller manager making requests to the API server as well as `kube-probe` checking the health of the API server.

From a security standpoint, whilst this is no replacement for audit logging, there is some interesting data there, although in production it's worth remembering that traces would likely be sampled and not 100% of them would be sent to the backend.

## Setting up Docker with OTel

There's also support for tracing in Docker, which is enabled by adding environment variables to the service. If you've got Docker running under systemd, you can edit the service file to add the environment variables. 

```bash
sudo systemctl edit docker.service
```

And then add the environment variables to the service file (replace 192.168.41.107 with the IP of your collector)

```bash
[Service]
Environment="OTEL_EXPORTER_OTLP_ENDPOINT=http://192.168.41.107:4318"
```

After that you can do a daemon-reload and then restart the service and you'll get traces showing up in your backend(s).

## Conclusion

As Open Telemetry uptake increases, it's likely that many services that we use will get support for it, enabling a standardized approach to observability instrumentation to be established. From a security standpoint, this has quite a bit of promise for improving our access to security information generated by applications, so it'll be interesting to see how it develops. 