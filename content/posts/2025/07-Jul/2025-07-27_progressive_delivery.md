+++
title = 'Progressive Delivery using Argo Rollouts'
summary = 'Gradually and safely update Kubernetes workloads using advanced deployment strategies'
tags = ["progressive delivery", "kubernetes", "argo rollouts"]
date = 2025-07-27
showToc = true
draft = false
+++

## Progressive Delivery

Progressive delivery refers to the process of gradually releasing software updates. With Kubernetes (K8s), there's limited options to upgrade **Deployment** workloads and the options are potentially disruptive.

Out of the box, K8s `Deployment` workload offers 2 upgrade strategies [1]:
- **Recreate**: wait for all existing pods to stop and then deploy new pods
- **RollingUpdate** (default): gradually undeploy existing pods and deploy new pods at the same time until all existing pods are replaced

The `Recreate` strategy causes downtime because new pods are not deployed until all existing pods are stopped. The `RollingUpdate` strategy avoids downtime by simultaneously provisioning new pods as it is shutting down existing ones. However, the workload doesn't have many knobs and dials to control the rate at which pods are rolled. Finally, neither upgrade strategy gracefully handles rollbacks. To revert a bad deploy, you need to manually revert the manifests and trigger yet another upgrade. K8s will not automatically revert to the last good version.

## Argo Rollouts

Argo Rollouts is a K8s controller that can be installed in a K8s cluster [2]. Once installed, it introduces several new custom resource definitions (CRD). Today, we'll focus on the **Rollouts** resource.

The Rollouts resource is a new workload that behaves very similarly to a K8s Deployment [3]. As with a Deployment, a Rollout manages the lifecycle of **ReplicaSets** to faciliate upgrades and rollbacks. A Rollout will not automatically manage the replica sets of an existing Deployment unless you configure it to do so. This can be useful when you want to upgrade a specific Deployment to a Rollout.

The key difference is that a Rollout supports advanced deployment strategies. It also offers optional features such as integrating with metric providers (Prometheus, Datadog) or ingress controllers/service meshes (NGINX, Istio). Integrating with metric providers allows Argo Rollouts to query metrics to automate the progress or rollback of a progressive upgrade. Integrating with an ingress controller or service mesh allows Argo Rollouts to achieve fine-grained traffic shifting during a progressive upgrade. We'll explore each of these features in follow-up posts.

## Advanced Deployment Strategies

Please note that a Rollout does not support Recreate or RollingUpdate upgrade strategies. Use a Deployment if you want prefer those strategies. Argo Rollouts is also not suitable for extended preview environments. If you want to deploy multiple versions for days or weeks, use another solution. Argo Rollouts is meant for upgrades that take minutes or hours. 

A Rollouts resource supports 2 upgrade strategies [4]:
- **Blue-Green**: new pods are deployment alongside existing pods; traffic will fully shift from the old pods to the new pods once the new pods are stable and then old pods will be shut down
- **Canary**: traffic is gradually shifted from old pods to new pods, in a controlled fashion, until all traffic is exclusively served from the new pods

### Blue-Green upgrades

The **Blue-Green** strategy is a marked improvement compared to the Recreate strategy because it avoids application downtime. The existing pods are kept at the desired count and continue to serve traffic. All traffic will then be shifted to the new pods once the new pods are ready. However, this strategy is still disruptive because 100% of the traffic is either directed to the old pods or new pods. The blast radius is huge if there's a business logic issue in the new version. Otherwise, if the new pods are unstable, Argo Rollouts will automatically revert to the last stable replica set.

Next, let's talk about the canary upgrade strategy.

### Canary upgrades

The **Canary** strategy supports a multi-phase upgrade in which traffic is gradually shifted from the `stable` replica set to the `canary` replica set. The basic version of this strategy does not require an external traffic manager such as an Ingress controller or a service mesh. However, you lose the ability to configure percentage-based traffic shifting policies in the absence of a traffic manager.

In its basic form, the Rollouts controller is capable of shifting traffic at a coarse-grained interval using pod counts. For example, if the application has a desired pod count of 10, then 10% of traffic can be shifted at a time by provisioning one canary pod and deprovisioning one stable pod. If the app has deployed 100 pods, we can achieve up to 1% shifts of traffic at a time.

Unlike a RollingUpdate, you can control the precise rate at which traffic is shifted over from the stable set to the canary set. Let's examine how in the example in the next section.

## Canary rollout example
Here's an example of a Rollouts manifest that utilizes a canary upgrade strategy:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
    name: my-app-rollout
spec:
    replicas: 20
    selector:
        matchLabels:
            app: my-app
    workloadRef:
        apiVersion: apps/v1
        kind: Deployment
        name: my-app-deployment
        scaleDown: onsuccess
    progressDeadlineSeconds: 600
    progressDeadlineAbort: false
    strategy:
        canary:
            stableService: my-app-stable-svc
            canaryService: my-app-canary-svc
            stableMetadata:
                labels:
                    role: stable
            canaryMetadata:
                labels:
                    role: canary
            scaleDownDelaySeconds: 60
            steps:
                - setCanaryScale:
                    matchTrafficWeight: true
                - pause:
                    duration: 30s
                - setWeight: 1
                - pause: {}
                - setWeight: 35
                - pause:
                    duration: 3m
                - setWeight: 65
                - pause:
                    duration: 3m
```

Let's break down what's happening in this manifest. First, the top-level resource kind is `Rollout`. As a reminder, a Rollout can be used as a drop-in replacement for deployments. In the snippet below, we have an existing Deployment that we want to upgrade to a Rollout, so we can make use of the `spec.workloadRef` field to reference the Deployment's pod template. The `spec.workloadRef.scaleDown` field allows us to specify whether to stop the Deployment pods once it's been replaced by Rollout pods. Alternatively, we can eschew the `spec.workloadRef` and simply define the pod template directly using `spec.template` field. Do note this field is mutually exclusive with the `spec.workloadRef` field. **Do not specify both!**

```yaml
workloadRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app-name-deployment
    scaleDown: onsuccess
```

Next, we configure the `spec.strategy` field to use the canary upgrade strategy. Since we have not integrated a traffic manager, we'll use k8s Service resources to shift traffic from the stable pods to the canary pods. This is done with the `spec.strategy.canary.stableService` and `spec.strategy.canary.canaryService` configs. The `spec.strategy.canary.stableMetadata` config is used to annotate and/or label the stable pods to help you identify it. However, `spec.strategy.canary.canaryMetadata` annotations/labels are ephemeral since canary pods only exist during an active upgrade.

Finally, traffic will be shifted over using relative pod counts. Because we've set a desired pod count of 20, this means we can shift traffic in increments of 5% assuming an even distribution. If we want arbitrary percentages, we need to use an ingress controller or service mesh.

```yaml
strategy:
    canary:
        stableService: my-app-name-stable-svc
        canaryService: my-app-name-canary-svc
        stableMetadata:
            labels:
                role: stable
        canaryMetadata:
            labels:
                role: canary
```

Finally, the `spec.strategy.steps` field is where we define a canary upgrade strategy.

```yaml
steps:
    - setCanaryScale:
        matchTrafficWeight: true
    - setWeight: 5
    - pause: {}
    - setWeight: 35
    - pause:
        duration: 3m
    - setWeight: 65
    - pause:
        duration: 3m
```

By default, the canary replica pod count is scaled to the amount of traffic that it handles (`setCanaryScale.matchTrafficWeight: true`). If 35% of traffic is shifted to the canary pods, then the canary pod count will be 35% of the desired count whereas the stable pods will be lowered to 65%. In this way, K8s distributes traffic to the stable and replica set based on the relative pod counts in each. This is what the `matchTrafficWeight: true` setting does.

The Rollout phases:
1. First, 5% of traffic is shifted to the canary pods. Given our desired pod count of 20, this means there'll be 19 stable pods and 1 canary pod
2. The rollout pauses indefinitely to give the operator time to review metrics, logs, and alerts. If the operator is satisfied, they can promote the rollout to the next step.
3. 35% of traffic is shifted to the canary. There's now 13 stable pods and 7 canary pods.
4. The rollout pauses for 3 minutes and then automatically proceeds to the next step
5. 65% of traffic is now shifted to the canary
6. Pause for 3 minutes
7. Complete the rollout by shifting the remaining traffic to the canary replica set
8. Wait `spec.strategy.canary.scaleDownDelaySeconds` before shutting down the stable pods
9. Perform some clean-up such as removing the `spec.strategy.canary.canaryMetadata` annotations/labels and replacing them with the `spec.strategy.canary.stableMetadata` annotations/labels
10. The Rollout is now complete and the canary replica set is marked as the latest stable replica set

If you're in doubt, start with the BlueGreen strategy. Then, try the canary strategy. Finally, explore fully automated upgrades using the `Analysis` CRD and tools like Kayenta.

## Conclusion

Kubernetes offers limited options to upgrade a Deployment. `Recreate` results in downtime and `RollingUpdate` offers limited ways to control progress/rate of the upgrades. Neither strategy gracefully handles rollbacks.

Argo Rollouts is a controller that can be installed in a K8s cluster. It introduces the `Rollouts` workload resource which behaves similarly to a `Deployment`. It supports `Blue-Green` and `Canary` upgrade strategies. If the upgrade fails, Argo Rollouts will automatically rollback the deployment to the last stable replica set.

In a future article, we'll discuss more Argo Rollout's advanced features including fine-grained traffic shifting and metric analysis.

## References

* [1] https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy
* [2] https://argoproj.github.io/argo-rollouts/
* [3] https://argoproj.github.io/argo-rollouts/features/specification/
* [4] https://argoproj.github.io/argo-rollouts/concepts/#deployment-strategies
