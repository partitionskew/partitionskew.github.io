+++
title = 'Istio - Locality-based Load Balancing'
summary = 'What is locality based load balancing? When should it be enabled or disabled?'
tags = ["istio", "service mesh", "load balancing"]
date = 2025-03-16
showToc = true
draft = false
+++

## Introduction

[Istio](https://istio.io/latest/about/service-mesh/) is an open-source service mesh with rich features and a steep learning curve. It's an infrastructure layer deployed in a Kubernetes cluster to handle networking responsibilities such as mutual TLS, load balancing (LB), and traffic shifting. It offloads this responsibility from distributed applications which can then focus on core business logic. It's analogous to the sidecar design pattern which offloads cross-cutting features such as logging or metrics from the app.

Istio provides configurable traffic management through its various custom resource definitions (CRDs). In this article, we discuss the locality-based load balancing feature and when it should be used.

## Load Balancing

The `DestinationRule` CRD enables operators to define policies for traffic that is routed to a particular destination in the mesh. A key configuration is the load balancing algorithm. By default, Istio uses the `least requests` algorithm to route traffic to the destination instances [1].

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: dest-svc-destination-rule
spec:
  host: dest-svc.prod.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST
```

It is generally recommended to use the default setting as it produces the best results out of the box. 

## Locality based load balancing

We can further augment our traffic routing with locality-aware load balancing. First, a brief refresher on availability zones.

### Availability Zones

To achieve high availability (HA), applications are frequently deployed to multiple availability zones (AZ) within a given region. If one zone goes down, the service continues to serve traffic since the other zones are geographically isolated in the same region [2].

A very marginal benefit is low latencies since traffic stays in a single AZ. However, AZs in a region are typically connected by redudant, high speed fiber. Cloud platforms generally strive for single-digit millisecond latencies across zones [3].

The more significant benefit is that network traffic that stays in a given zone does not incur data transfer costs. Cross-zonal network transfer costs can be non-trivial for high volume services. In the case of Kafka, it can account for as much as 88% of overall costs [4]. Hence, it's an easy-to-overlook, but important, source of operational costs.

### Locality Based Load Balancing

By default, Istio enables locality balanced load balancing in the global mesh configuration [5].

It can be selectively overridden for a particular service using the `DestinationRule` CRD mentioned earlier.

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: dest-svc-destination-rule
spec:
  host: dest-svc.prod.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST
      localityLbSetting:
        enabled: true
        distribute:
          - from: us-west/zone1/*
            to:
              "us-west/zone1/*": 80
              "us-west/zone2/*": 20
          - from: us-west/zone2/*
            to:
              "us-west/zone1/*": 20
              "us-west/zone2/*": 80
```

As we can see, the load balancer algorithm and locality-based load balancing feature are not mutually exclusive. They are complementary. The latter affects the proportion of traffic that goes to a particular zone based on the originating zone. The former further refines the traffic routing to the various endpoints/instances within a given zone.

The example above is taken from the official reference documentation [6]. In the example, we configured 80% of traffic originating from a given zone to be forwarded to the same zone. The remaining 20% is routed to the other zone.

There are two other mutually exclusive strategies other than distribute:
* failover: traffic is routed to another region/zone when local endpoints are unhealthy
    * requires outlier detection to be enabled
* failoverPriority: similar to failover, but allows specification of priority among the failover endpoints

### Unequal traffic routing

One potential issue with locality-based load balancing is uneven distribution of traffic to the destination endpoints [7]. Imagine your destination service is deployed equally in 3 AZs. However, your source service may only be deployed in one of those zones. Alternatively, it may be deployed in all 3, but most pods are concentrated in one zone.

With locality-based load balancing, Istio prioritizes keeping traffic in the same zone. Consequently, the destination pods are overwhelmed with the majority of the traffic whereas the other two zones are underutilized. CPU-based auto-scaling will not help since it averages the CPU usage across all 3 zones. The average CPU usage could very easily end up below the threshold for scaling out. The end result is your overloaded pods start responding to requests with higher latencies as they become CPU-starved. In the worst case, they may even fail.

![locality_based_load_balancing](/images/2025-03-15_locality_lb.png)
**Figure 1: Average requests/second received by destination pods in each zone.**

The graph above depicts the average request rate to destination pods grouped by the AZ. Since most of the source pods are deployed in zone C, most of the traffic is routed to destination pods in zone C due to locality-based load balancing. The average request rate per pod in zone C is several times higher than pods in the other two zones.

Upon disabling locality-based load balancing, the average request rate evens out across the zones. The trade-off is more cross-zonal network transfer and ever-so-slightly higher latencies. This trade-off is worth making if you have no control over the source deployment. If we could adjust the zonal distribution of source pods, then we should ensure the two mirror each other to avoid skew.

## Conclusion

Cross-zonal network transfer is a hidden, but hefty, charge when working with multi-zonal applications. Istio enables locality-based load balancing by default to ensure traffic stays within the same zone. However, Istio offers users the ability to fine-tune the distribution, only enable it during failover, or disable it altogether.

When the source and destination zonal pod distributions do not match each other, it can lead to unequal traffic routing. In such cases, the distributions should either be adjusted to mirror each other or locality-based load balancing should be disabled altogether.

Finally, locality-based load balancing is complementary to regular load balancing algorithms such as random, round-robin, or least requests. They work in conjunction to achieve optimal routing.

## References

* [1] https://istio.io/latest/docs/concepts/traffic-management/#load-balancing-options
* [2] https://en.wikipedia.org/wiki/Availability_zone
* [3] https://learn.microsoft.com/en-us/azure/reliability/availability-zones-overview?tabs=azure-cli#inter-zone-latency
* [4] https://www.confluent.io/blog/understanding-and-optimizing-your-kafka-costs-part-1-infrastructure/#networking
* [5] https://istio.io/latest/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-locality_lb_setting
* [6] https://istio.io/latest/docs/reference/config/networking/destination-rule/#LocalityLoadBalancerSetting
* [7] https://github.com/istio/istio/issues/37231
