---
layout: post
title: "Taking a look at the Kube-Proxy API" 
date: 2024-06-16 12:27:00 +0100
comments: false
categories: Kubernetes
---

Kubernetes has got a number of different components, each with it's own API. Whilst most of the time you'll interact with the main kube-apiserver API, and sometimes the Kubelet API, the other ones can have some interesting properties. The kube-proxy API is interesting, in that it has some differences from all the others. 

The API is split into two separate components, the `healthz` API and the `metrics` API. The healthz API, which listens on `0.0.0.0:10256` by default is extremely simple, having one endpoint `/healthz`. It doesn't have any option for authentication, so you just request that endpoint and you get a response (N.B. Like a lot of Kubernetes APIs if you request the root path you'll get a 404).

```bash
curl http://127.0.0.1:10256/healthz
{"lastUpdated": "2024-06-16 13:10:38.185046097 +0000 UTC m=+18599.921395918","currentTime": "2024-06-16 13:10:38.185046097 +0000 UTC m=+18599.921395918", "nodeEligible": true}
```

The response is kind of interesting as it provides some time and other metadata, unlike the other components which just return a flat `ok` to requests to `healthz`

The `metrics` API has a default bind address of `127.0.0.1:10249` and has some more interesting endpoints available. Unlike other APIs in Kubernetes, there's no authentication option for this service so anyone who can reach it, can access any endpoint. Also note that the bind address being localhost is a distribution choice. For example Amazon EKS binds this service to all interfaces (after reporting this to them I was told this is by design and pointed at [this GitHub issue](https://github.com/aws/containers-roadmap/issues/657)).

The `/metrics` endpoint returns a large list of information about the host and cluster's metrics, in Prometheus format. One thing that caught my eye when looking through the output is that it provides information on what Alpha and beta features are enabled by the cluster. I'm not sure why this information is included in a Node component API, but if you're surveying a cluster (particularly a managed k8s cluster where you don't have access to the control plane) it could be of interest.

An excerpt of the output about features looks like this

```
kubernetes_feature_enabled{name="APIListChunking",stage=""} 1
kubernetes_feature_enabled{name="APIPriorityAndFairness",stage=""} 1
kubernetes_feature_enabled{name="APIResponseCompression",stage="BETA"} 1
kubernetes_feature_enabled{name="APIServerIdentity",stage="BETA"} 1
kubernetes_feature_enabled{name="APIServerTracing",stage="BETA"} 1
kubernetes_feature_enabled{name="APIServingWithRoutine",stage="BETA"} 1
kubernetes_feature_enabled{name="AdmissionWebhookMatchConditions",stage=""} 1
kubernetes_feature_enabled{name="AggregatedDiscoveryEndpoint",stage=""} 1
kubernetes_feature_enabled{name="AllAlpha",stage="ALPHA"} 0
kubernetes_feature_enabled{name="AllBeta",stage="BETA"} 0
kubernetes_feature_enabled{name="AllowServiceLBStatusOnNonLB",stage="DEPRECATED"} 0
kubernetes_feature_enabled{name="AnyVolumeDataSource",stage="BETA"} 1
kubernetes_feature_enabled{name="AppArmor",stage="BETA"} 1
kubernetes_feature_enabled{name="AppArmorFields",stage="BETA"} 1
kubernetes_feature_enabled{name="CPUManager",stage=""} 1
kubernetes_feature_enabled{name="CPUManagerPolicyAlphaOptions",stage="ALPHA"} 0
```

Another interesting endpoint is `/configz` this one returns the configuration of the component without any authentication. The example below comes from a Kubeadm cluster and as you can see there's some information disclosure including physical paths.

```json
{
  "kubeproxy.config.k8s.io": {
    "FeatureGates": {},
    "ClientConnection": {
      "Kubeconfig": "/var/lib/kube-proxy/kubeconfig.conf",
      "AcceptContentTypes": "",
      "ContentType": "application/vnd.kubernetes.protobuf",
      "QPS": 5,
      "Burst": 10
    },
    "Logging": {
      "format": "text",
      "flushFrequency": "5s",
      "verbosity": 0,
      "options": {
        "text": {
          "infoBufferSize": "0"
        },
        "json": {
          "infoBufferSize": "0"
        }
      }
    },
    "HostnameOverride": "kind-control-plane",
    "BindAddress": "0.0.0.0",
    "HealthzBindAddress": "0.0.0.0:10256",
    "MetricsBindAddress": "127.0.0.1:10249",
    "BindAddressHardFail": false,
    "EnableProfiling": false,
    "ShowHiddenMetricsForVersion": "",
    "Mode": "iptables",
    "IPTables": {
      "MasqueradeBit": 14,
      "MasqueradeAll": false,
      "LocalhostNodePorts": true,
      "SyncPeriod": "30s",
      "MinSyncPeriod": "1s"
    },
    "IPVS": {
      "SyncPeriod": "30s",
      "MinSyncPeriod": "0s",
      "Scheduler": "",
      "ExcludeCIDRs": null,
      "StrictARP": false,
      "TCPTimeout": "0s",
      "TCPFinTimeout": "0s",
      "UDPTimeout": "0s"
    },
    "Winkernel": {
      "NetworkName": "",
      "SourceVip": "",
      "EnableDSR": false,
      "RootHnsEndpointName": "",
      "ForwardHealthCheckVip": false
    },
    "NFTables": {
      "MasqueradeBit": 14,
      "MasqueradeAll": false,
      "SyncPeriod": "30s",
      "MinSyncPeriod": "1s"
    },
    "DetectLocalMode": "ClusterCIDR",
    "DetectLocal": {
      "BridgeInterface": "",
      "InterfaceNamePrefix": ""
    },
    "ClusterCIDR": "10.244.0.0/16",
    "NodePortAddresses": null,
    "OOMScoreAdj": -999,
    "Conntrack": {
      "MaxPerCore": 0,
      "Min": 131072,
      "TCPEstablishedTimeout": "24h0m0s",
      "TCPCloseWaitTimeout": "1h0m0s",
      "TCPBeLiberal": false,
      "UDPTimeout": "0s",
      "UDPStreamTimeout": "0s"
    },
    "ConfigSyncPeriod": "15m0s",
    "PortRange": ""
  }
}
```

## Conclusion

This was just a quick note with a look at the kube-proxy API. Of the APIs that Kubernetes presents, it's probably not the most interesting from a security perspective, but still has some interesting information disclosure and the choice to not provide authentication does make it an interesting target for reconnaissance.