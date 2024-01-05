When working with vSphere with Tanzu on NSX-T, the cluster-to-cluster communication between Tanzu Kubernetes Clusters on the same vSphere Workload Management Cluster was observed to deviate from other combinations of communication.

This means that we cannot use the Tanzu Supervisor Namespace Egress IP as the source IP when we create Gateway Firewall rules to allow cluster-to-cluster communication.

This is due to the default NAT rules that NSX-T creates when a new Supervisor Namespace is created:

<p align="center">
  <picture>
    <img alt="NSX-T NAT Rules" src="https://github.com/sachua/sachua.github.io/assets/41395198/aef1aaa7-219d-486b-ae25-ea1f73bcc490" width="1000">
  </picture>
</p>

A quick look at the 3 NAT rules set:

1. **No SNAT for traffic going from Node IPs (10.222.0.0/16) to Ingress IPs (10.223.112.0/20)**

    - No SNAT is applied to any traffic going from Cluster A to Cluster B
 
2.  **No SNAT for traffic going from Node IPs (10.222.0.0/16) to Node IPs (10.222.0.0/16)**

    - No SNAT is applied to any traffic going from Node A to Node B

3. **SNAT for all other traffic, translated to the Tanzu Supervisor Namespace Egress IP (10.223.136.18)**

    - SNAT is applied to any other traffic, such as from Cluster A to another Virtual Machine in another segment, and egress the Tanzu Supervisor Namespace segment through the Tanzu Supervisor Namespace Egress IP

## What does this mean?

**Rule 1:**

<p align="center">
<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://github.com/sachua/sachua.github.io/assets/41395198/57b57f99-1da9-4a0a-9430-3986f0e30c00">
  <img alt="Rule 1 illustration" src="https://github.com/sachua/sachua.github.io/assets/41395198/0c048f66-4e49-4d6c-9865-af9d66108a48" width="1000">
</picture>
</p>

For Cluster-to-Cluster traffic, SNAT rule 1 is applied, Cluster B will see the incoming traffic IP address as the Node IP of Cluster A

**Rule 2:**

<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://github.com/sachua/sachua.github.io/assets/41395198/897e4a6d-b4bd-40f4-9b34-2cc910d19a35">
    <img alt="Rule 2 illustration" src="https://github.com/sachua/sachua.github.io/assets/41395198/527e70b3-0586-4bc8-8408-53750e2b1272" width="350">
  </picture>
</p>

For traffic within the Cluster, SNAT rule 2 is applied, Nodes within Cluster A will see each other's IP address. This is working as intended.

**Rule 3:**

<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://github.com/sachua/sachua.github.io/assets/41395198/ea16758c-a8c7-4546-84ba-7321134b0ce0">
    <img alt="Rule 3 illustration" src="https://github.com/sachua/sachua.github.io/assets/41395198/fe45e3b0-bc7b-4247-8a14-d71a74fd1dae" width="800">
  </picture>
</p>

For Cluster-to-Virtual-Machine traffic, SNAT rule 3 is applied, the Virtual Machine will see the incoming traffic IP address as the Tanzu Supervisor Namespace Egress IP which Cluster A resides in. This is working as intended.

<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://github.com/sachua/sachua.github.io/assets/41395198/9aeb12f6-60da-471e-9ab4-66d3cc2666e7">
    <img alt="Rule 3 illustration" src="https://github.com/sachua/sachua.github.io/assets/41395198/b31b002c-d570-4758-90f6-8a51def9105d" width="700">
  </picture>
</p>

In fact, for network traffic that leaves the T0 Gateway to an external Kubernetes cluster, the external Kubernetes cluster will also see the incoming traffic IP address as the Tanzu Supervisor Namespace Egress IP which Cluster A resides in. This is also working as intended.

## Hypothesis

Because of the implementation of vSphere Pods, where the Tanzu Supervisor Namespaces are treated as Kubernetes namespaces, and vSphere Pods are treated as Kubernetes pods, the network communication between Kubernetes pods in different namespaces should show the internal IP addresses and not the Tanzu Supervisor Namespace egress IP. In this implementation, SNAT Rule 1 is valid.

The problem happens when SNAT Rule 1 is retained in NSX-T even after the Tanzu Kubernetes Cluster deployment model was selected, where the Tanzu Kubernetes Cluster nodes should not exist in the same segment as other Tanzu Kubernetes Cluster nodes when they exist in different Tanzu Supervisor Namespaces.

## Workaround

Since we cannot use the Tanzu Supervisor Namespace Egress IP as the source, we need to find a way to obtain the IP address of the Cluster Nodes. 

If we do some exploring, we observe there are some default NSX-T Distributed Firewall rules that are created whenever a new Supervisor Namespace is created to allow intra-cluster communication by default.

<p align="center">
  <picture>
    <img alt="Auto-generated security group in NSX-T Distributed Firewall for Supervisor Namespace" src="https://github.com/sachua/sachua.github.io/assets/41395198/74241e7d-d9ac-4545-b74c-9b1d5a6a2759" width="1000">
  </picture>
</p>

These rules are applied to 2 security groups, by clicking the security groups, we can see that one of the security groups contain all Cluster Nodes that exist within the Tanzu Supervisor Namespace.

<p align="center">
  <picture>
    <img alt="Members in auto-generated security group" src="https://github.com/sachua/sachua.github.io/assets/41395198/6d197148-f4ad-4899-9f96-2c087a4143af" width="500">
  </picture>
</p>

We can then select this security group as our source when creating Gateway Firewall rules, and the network communication will work as desired.

The downside is this solution does not have the ability to set fine-grained rules for Tanzu Kubernetes Clusters that exist in the same Tanzu Supervisor Namespace, as the security group contains everything.

This means for Cluster A and Cluster B, both existing in the same Tanzu Supervisor Namespace, we cannot allow traffic from Cluster A but deny traffic from Cluster B.

<!-- ##{"timestamp":1704440241,"script":"<script>document.querySelectorAll('picture').forEach((picture)=>{picture.querySelectorAll(`source[media*="prefers-color-scheme"],source[data-media*="prefers-color-scheme"]`).forEach((source)=>{if(source?.media.includes('prefers-color-scheme')){source.dataset.media = source.media;}if(source?.dataset.media.includes(localStorage.getItem("meek_theme"))){source.media='all';}else if(source){source.media='none';}})})</script>"}## -->