---
layout: post
title:  "Deploying Ansible AWX on K3s"
date:   2023-02-14 19:30:00 -0500
categories: awx kubernetes ansible
published: true
---
Working with Ansible Automation Platform daily is wonderful.  I get to see the value of the offering's potential and value for our customers, and how powerful an upstream solution and community can be when delivering an enterprise product.

When I started working at Red Hat, I was familiar with Ansible CLI tools, but I wasn't familiar with Ansible Automation Controller, Ansible Automation Hub, nor the differences between virtual machine and container based deployment options.  So, I set out deploying AWX on my K3s cluster so that I could experiment with the upstream to learn about the platform, and to attempt to put myself in the shoes of a potential customer that may be considering AWX versus Ansible Automation Platform.

You might ask why this article uses K3s instead of OpenShift or OKD; it's because I have had a K3s cluster running for years and have not attempted to migrate my homelab.  That's next!

## Installing the Operator

The AWX operator has gone through some significant improvements since I first started this effort.  Now, a `kustomize` command can deploy the operator.  But, that wasn't always the case.  Deploying the operator was dependent on `amd64` tools, and I was on an M1 Mac.

So, I packaged the deployment dependencies until a [container](https://quay.io/repository/scottharwell/m1-awx-operator-runner) that could be rebuilt to support `arm64` and `amd64` architectures.  The purpose of the container was just to deploy the operator, so its entrypoint is set to the command(s) to make that happen.  Even now, with kustomize as the deployment preference, I create this container with each new AWX release so that I can run a single command to deploy the operator.  It uses the [the same kustomize command](https://github.com/scottharwell/m1-awx-operator-runner/blob/main/Dockerfile) that AWX docs instruct, but packages the steps into a single command that can be run.

On each new AWX release, I run the following command, and AWX is then upgraded within the cluster (it does take a few minutes for the operator to tear down the old version and deploy the new version).

```bash
docker run \
-it \
--rm \
--name awx-operator-runner \
--pull=always \
--mount type=bind,source="$HOME/.kube/config",target=/home/awx/.kube/config \
quay.io/scottharwell/m1-awx-operator-runner:latest
```

## Deploying AWX

The following configuration file is the result of many tweaks and edits to tune specifically for the limited resources of my cluster and my limited automation needs.  At first, I just wanted to get AWX running, and that required understanding the different ingress options that the AWX operator manages and how they would interact with Traefik, which is the standard ingress on K3s.  Though, with a few ingress annotations for Traefik and CertManager, the ingress works exactly as expected.

* I have a middleware in the default namespace that redirects all HTTP traffic to HTTPS
* I expose both HTTP and HTTPS ports so that browsers can be redirected if HTTP is requested
* I use a cluster issuer to request an SSL certificate from CertManager

I chose to use block storage through the Longhorn storage driver that I have installed on K3s.  I also have an NFS storage provisioner, but given the limited storage needed for this deployment, I chose to use Longhorn for both the database and project storage.  Though, project storage could reside on NFS.

I set the timezone for each of the container types so that timezone accountability is consistent across the platform, and that time is reported as my local timezone.

The remaining configuration was changing and testing the resource requirements for the individual services that the operator manages.  And, as of today, I find the current configuration keeps the UI snappy, automation running, and upgrades are smooth.

```yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: harwell-awx
  namespace: awx
spec:
  service_type: clusterip
  ingress_type: ingress
  hostname: awx.domain.name
  ingress_annotations: |
    environment: awx
    traefik.ingress.kubernetes.io/router.middlewares: default-redirect@kubernetescrd
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
    cert-manager.io/cluster-issuer: letsencrypt-aws
  ingress_tls_secret: tls-cert
  ingress_path: /
  ingress_path_type: Prefix
  postgres_storage_class: longhorn
  postgres_keep_pvc_after_upgrade: false
  postgres_resource_requirements:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 2000m
      memory: 1Gi
  postgres_storage_requirements:
    requests:
      storage: 2Gi
    limits:
      storage: 2Gi
  web_resource_requirements:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 1Gi
  task_resource_requirements:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 2000m
      memory: 2Gi
  ee_resource_requirements:
    requests:
      cpu: 500m
      memory: 256Mi
    limits:
      cpu: 2000m
      memory: 2Gi
  redis_resource_requirements:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 1Gi
  projects_persistence: true
  projects_storage_size: 5Gi
  projects_storage_class: longhorn
  projects_storage_access_mode: ReadWriteMany
  image_pull_policy: IfNotPresent # Always # for AWX-ee image
  ee_images:
    - name: Harwell - Cloud EE
      image: quay.io/scottharwell/cloud-ee:latest
  ipv6_disabled: false
  ee_extra_env: |
    - name:  TZ
      value: America/New_York
  task_extra_env: |
    - name:  TZ
      value: America/New_York
  web_extra_env: |
    - name:  TZ
      value: America/New_York
```

Try running AWX in your home lab.  It's a great learning experience for both Kubernetes and Ansible technologies.
