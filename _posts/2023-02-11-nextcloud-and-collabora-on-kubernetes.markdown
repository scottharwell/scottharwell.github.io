---
layout: post
title:  "Running Nextcloud and Collabora CODE in Kubernetes"
date:   2023-02-11 11:50:00 -0500
categories: kubernetes nextcloud collabora
published: true
---
This is the introduction to my series of posts regarding [running Nextcloud and Collabora CODE in Kubernetes][root-post].

1. [Deploying Nextcloud in Kubernetes][nextcloud-post]
2. [Deploying Collabora CODE in Kubernetes][collabora-post]
3. [Configuring Nextcloud with Collabora Code in Kubernetes][configuration-post]

---

Over the last month, I have been experimenting with running [Nextcloud][nextcloud] and [Collabora CODE][collabora] (for Nextcloud Office) as a replacement for Office 365.  I have a commercial Office 365 account, but I don't use it enough to justify the price.  And, rather than use a personal account from Microsoft or Google, I wanted to explore hosting my own cloud services.

I have run [Nextcloud][nextcloud] and ownCloud servers in the past. But, I have always deployed them directly to VMs.  I run a K3s Kubernetes cluster in my home lab, and so I want to run all of the services that I can from my cluster in order to share the resources of the cluster and to make orchestration across many applications easier.

TL;DR I have both [Nextcloud][nextcloud] and [Collabora CODE][collabora] running in my cluster, and the performance is on-par with my previous VM-based deployments.  Though, it took some experimentation to get to this state.  This article covers the process to get to my successful configuration.

## Upfront Gotchas

### Collabora CODE and DNS

I found that Collabora CODE would sometimes stop responding when private DNS was used.  I could not determine the cause, because the services in the pod would function, then simply stop functioning without any indicator as to why in the logs.  And, redeploying the pod would work for a time, and then repeat the issue.  My cluster is directly connected to my private DNS server, so lookups to private records should not fail.  

I stopped using private DNS and started using public DNS to determine if DNS was the issue. The problems stopped.  I'm not convinced that this is an issue with Collabora's DNS lookups; it could be a CoreDNS issue or something at the cluster level.  But, I have many other applications running with private DNS and Collabora is the only service that exhibited this behavior.  I've kept public DNS for now and locked ingress to private IPs to secure the Collabora service.

If you're only deploying Nextcloud and do not intend to use Nextcloud Office, then using a private DNS record without exposure to ingress from the public internet works just fine.  This was only an issue for the NextCloud office part of the deployment.

## Cluster Configuration

I have a three-node [K3s][k3s] cluster running on [Red Hat Enterprise Linux][rhel] virtual machines.  Each VM has 16 GB of RAM.  The primary cluster node has eight CPU cores assigned, and the two worker nodes have four CPU nodes assigned.

I'll probably cover installing and configuring [K3s][k3s] on RHEL in a different article, but once installed, despite setting the [SELinux options][selinux], the RHEL nodes still reported issues in the audit logs.  After running `audit2allow` to create policies that allowed the blocked processes, many performance issues and disconnects stopped occurring.

I mention SELinux at the start of the article since later errors may be cause by a need to create an allow policy.

I also have two storage types setup on my cluster, [Rancher Longhorn][longhorn] and [NFS client storage][nfs-storage].  I use the block storage provided by Longhorn for database and configuration storage since it's much faster than the NFS storage.  However, Nextcloud's file storage is backed by the NFS provisioner.  Originally, I tried using Nextcloud's object storage with Minio, but I found that the file transfer performance was halved just using Minio from the same TrueNAS server that my NFS share was mounted.

I use K3s' default Traefik ingress, and I have installed CertManager to manage SSL certificates.

## Articles

The configuration examples for Nextcloud and Collabora CODE are lengthy, so I have broken the deployment into steps.

1. [Deploying Nextcloud in Kubernetes]({% post_url 2023-02-11-deploying-nextcloud-in-kubernetes %})
1. [Deploying Collabora CODE in Kubernetes]({% post_url 2023-02-11-deploying-collabora-code-in-kubernetes %})
1. [Configuring Nextcloud with Collabora Code in Kubernetes]({% post_url 2023-02-11-configuring-nextcloud-and-collabora-on-kubernetes %})

[rhel]: https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux
[nextcloud]: https://nextcloud.com/
[collabora]: https://www.collaboraoffice.com/code/
[k3s]: https://docs.k3s.io/installation
[selinux]: https://docs.k3s.io/advanced#selinux-support
[longhorn]: https://docs.k3s.io/storage
[nfs-storage]: https://www.phillipsj.net/posts/k3s-enable-nfs-storage/
[root-post]: {% post_url 2023-02-11-nextcloud-and-collabora-on-kubernetes %}
[nextcloud-post]: {% post_url 2023-02-11-deploying-nextcloud-in-kubernetes %}
[collabora-post]: {% post_url 2023-02-11-deploying-collabora-code-in-kubernetes %}
[configuration-post]: {% post_url 2023-02-11-configuring-nextcloud-and-collabora-on-kubernetes %}
