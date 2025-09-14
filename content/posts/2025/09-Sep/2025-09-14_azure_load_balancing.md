+++
title = 'Azure - Load Balancing'
summary = 'A brief survey of the layer 4 and layer 7 load balancers in Azure'
tags = ["load balancing", "layer 4", "layer 7", "networking", "azure", "ingress"]
date = 2025-07-27
showToc = true
draft = false
+++

## Azure Load Balancing

Azure provides a plethora of load balancing (LB) options [1]. Some operate at the network layer (L4); others at the application layer (L7). Some load balancers are regional whereas others are global.

This article surveys the core LB offerings with a particular focus on the L4 Load Balancer and the L7 Application Gateways. We'll see that each solution has some trade-offs worth considering.

## Azure Load Balancer (L4)

This product is the standard network load balancer that routes traffic to a set of virtual machines (VMs) or a **virtual machine scale set** (VMSS). 

> FYI: VMSS is an Azure construct that enables you to group VMs together for easier management such as applying auto-scaling policies [2].

The Azure LB resides in your virtual network and can handle both inbound and outbound traffic. Moreover, the Azure LB supports public IP addresses, private IP addresses, or both. This enables you to configure a public or private LB.

The LB passes through network traffic which leads to low latency and high throughput. It's important to note that the client establishes a direct connection to the backend server. Depending on your organization's security policy, this can be a dealbreaker [3].

Conversely, it cannot terminate TLS connections since that's a Layer 7 concern. You'll need to combine the network load balancer with another solution, such as a service mesh, in order to terminate TLS and perform advanced traffic management.

Security is enforced with network security groups (NSG). By default, traffic is blocked unless permitted by a NSG on a subnet or network interface card (NIC). 

Finally, the public load balancer supports DDOS protection. Refer to the documentation for more info: https://learn.microsoft.com/en-us/azure/load-balancer/tutorial-protect-load-balancer-ddos

## Application Gateways (L7)

If you require more advanced traffic routing capabilities, you'll need to upgrade to a layer 7 load balancer. Azure doesn't provide a standard application load balancer (ALB) like Amazon or Google, but it does provide 2 gateway products that function similarly.

### Application Gateway V2

This product is available in most regions and offers many of the features that you expect from an ALB:
* auto-scaling
* can proxy layer 4 traffic (e.g. non-HTTP clients)
* regional with deployments spanning multiple zones for redundancy and high availability
* add, remove, or modify HTTP headers in the request/response
* supports URL rewrites
* integrates with Azure Kubernetes Service (AKS) through Application Gateway Ingress Controller [4]
* public and/or private frontend IP addresses

Unlike network load balancers, the Application Gateway V2 supports TLS termination and WAF security policies. It also supports DDOS protection for L3/L4 traffic.

The trade-off with the Application Gateway is that it does not support container-native load balancing. Unlike the Application Gateway for Containers, which we'll talk about next, traffic is forwarded to a VM and then Kubernetes routes the request to the backend pod which may or may not be on the same node. If the backend pod is not on the same host, then this results in a second hop and additional latency.

### Application Gateway for Containers

Application Gateway for Containers is also a layer 7 load balancer but view it as an evolution of Application Gateway V2 that better integrates with Kubernetes. Specifically, traffic is routed directly to the backend pods which reduces unnecessary hops and decreases latency [5]. This is known as container-native load balancing, a feature that is provided by Google ALBs as well [6].

This product also supports the K8s Gateway API, which enables more powerful traffic management rules to be expressed through Kubernetes resources compared to the Ingress API [7].

The trade-off with this product is that it's still relatively new and is not available in all regions. It also does not support private frontends. 

Nevertheless, it'd be my first choice for managing external traffic flow into a Kubernetes cluster.

## Azure Front Door (AFD)

Azure Front Door is a content delivery network (CDN) solution that is globally distributed. It minimizes client latency by connecting clients to the closest point of presence (PoP) which is deployed in key, strategic areas around the world [8].

AFD supports many of the same features as the Application Gateways including TLS termination, WAF security policies, and DDOS protection. 

AFD is particularly effective if your app benefits from caching content. However, AFD is not mutually exclusive with Application Gateways. You can combine AFD with Application Gateway or Azure LB. One reason is to minimize client latency by taking advantage of the global distribution of AFD PoPs. The PoPs routes traffic across Azure's network backbone to the second layer. The second layer of load balancers then handles traffic routing/management.

## Conclusion

Azure offers a plethora of load balancing options. For basic, non-HTTPS traffic, an Azure Load Balancer suffices. For primarily internet-facing, HTTPS traffic, consider using one of the two Application Gateways. If the backend is deployed in Azure Kubernetes Service (AKS), prioritize Application Gateway for Containers over Application Gateway V2. Finally, if you need global deployments and/or caching, use or combine Azure Front Door with the other load balancing products.

## References

* [1] https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/load-balancing-overview
* [2] https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-overview
* [3] https://learn.microsoft.com/en-us/azure/application-gateway/tcp-tls-proxy-overview#comparing-azure-load-balancer-with-azure-application-gateway
* [4] https://learn.microsoft.com/en-us/azure/application-gateway/ingress-controller-overview
* [5] https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/container-networking#cni-overlay-and-application-gateway-for-containers
* [6] https://cloud.google.com/kubernetes-engine/docs/concepts/container-native-load-balancing#architecture
* [7] https://gateway-api.sigs.k8s.io/
* [8] https://learn.microsoft.com/en-us/azure/frontdoor/edge-locations-by-region
