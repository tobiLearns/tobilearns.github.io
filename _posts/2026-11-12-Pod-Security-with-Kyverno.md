---
title: Pod Security with Kyverno
date: 2026-11-12
categories: [Homelab, Kubernetes]
tags: [talos, kubernetes]
---
In my last post, I described how I moved my Kubernetes cluster from k3s to Talos.
I learned that Talos enforces the *Pod Security Standards* profile `baseline` cluster-wide. 
k3s doesn't. I was happy about this extra security for free.

Then I saw that some workloads in my cluster, like MetalLB, the node-exporter in the monitoring stack, or the local-path-provisioner, 
need more than the `baseline` profile allows.
The *Pod Security Admission* (PSA) doesn't allow exceptions for single workloads.
Instead, an exception for a complete namespace is possible: the label `pod-security.kubernetes.io/enforce: privileged`.
But that switches off the checks for every Pod in the namespace, not just for the one that needs more rights. 
So this solution was a bit disappointing.

I heard about the tool *Kyverno*, a policy engine that can be used to close this gap again.
With Kyverno, I can define exceptions for single workloads (a "PolicyException"), and all other Pods keep being checked.
My first understanding was that Kyverno would replace the namespace label. But this wasn't the case.
PSA and Kyverno are independent checks, and a Pod has to pass both. 
This means a Kyverno exception can't override a PSA rejection. So the privileged namespaces had to stay privileged, 
but now Kyverno takes over the checking inside them.

To get the Pod security properly in place for all namespaces, I did the following:
1. Kept privileged namespaces as small as possible. 
   In my monitoring stack, only the node-exporter Pod needs access to the host (to read CPU load, storage and so on). 
   So I moved it into its own namespace, `monitoring-node-exporter`. 
   The `monitoring` namespace is back to the `baseline` profile, so Prometheus, Grafana and Alertmanager are protected again.
2. Let Kyverno check the privileged namespaces.
   Kyverno policies implementing the `baseline` profile now apply to every Pod there. 
   Only the workloads that really need more rights get a narrow, explicit Kyverno exception.
   For example, the node-exporter Pod may use `host namespaces` and `host paths`, but nothing else in its namespace may.
3. Added a `guard policy`. Only a set of namespaces, specified in an allow-list, may carry the privileged label. 
   A new or forgotten namespace can't end up unprotected without anyone noticing.

I rolled the policies out in `Audit` mode first and switched to `Enforce` after checking the policy reports.

So now all my workload namespaces are secured, and only very few single Pods have exceptions for some very specific rights.
Three namespaces are different: `kube-system` is excluded from PSA on Talos and from Kyverno. 
I also excluded `flux-system` and `kyverno` from Kyverno's checks, so Flux can always repair Kyverno. 
But these two namespaces are still protected by PSA.

How do you handle privileged workloads in your clusters?


