---
layout: post
title:  "Ansible - Cloud Execution Environment"
date:   2023-02-26 14:00:00 -0500
categories: ansible
published: true
---
Ansible Execution Environments (EEs) are a really powerful use of OCI compliant containers to package Ansible automation dependencies and run-time packages.  EEs inherit all of the power of containerization.  But, for those that are not familiar with containers, diving into EEs can be daunting as it requires basic knowledge of container technologies, like Docker or Podman, and the Ansible contents that ultimately get packaged into the EE.

About a year ago, I had both a personal and a professional need for an EE that I could use to run cloud automation.  At the time, I wanted to create a single EE that would allow me to run automation against all of the cloud providers that Ansible supports, including their CLI tools so that I could call the CLI if the cloud collection did not directly support my automation use case, such as Azure Arc.  So, I began work on my [Cloud EE][cloud-ee], which is an EE that accomplishes the following goals:

1. Packages the cloud collections that I need to use in a single EE.
2. Installs the CLIs for the major hyperscalers and their Python dependencies.
3. Runs on `amd64` and `arm64` architectures since I develop on a Mac but my automation runs on AWX or Ansible Automation Platform on `amd64` machines.

## Set Up

### Container Engine

In order to more easily package containers that would work across multiple architectures, I typically use Docker.  I plan to switch to Podman when it will support more robust volume mapping on a Mac, but for now, Docker is the easier option.

### Red Hat Developer Account

Access to a free [Red Hat developer account][rh-developer] will give you access to downstream Red Hat-built containers.  As long as you're not using the containers for commercial use cases, this is a great way to get access to officially supported container images.

You can build EEs with upstream container images, but sometimes they are not as stable as the downstream versions.  The [Cloud EE][cloud-ee] uses downstream containers.

### Container Registry

You'll need a container registry, such as [quay.io](https://quay.io) or [Docker Hub](https://docker.io) to push your container to when building.  Both offer free accounts for containers that are public.  If you have Ansible Automation Platform subscriptions, then you can use Private Automation Hub as your EE registry.  Any container registry that you have access to will ultimately work; public or private.

## The Execution Environment

There are a number of files that are used when building EEs.  For the [Cloud EE][cloud-ee], the following contain the important information:

| File                        | Description                                                                                                                                                                 |
| --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `execution-environment.yml` | Defines how the `Dockerfile` (for Docker) or `Containerfile` (for Podman) will be built.  It references all of the dependencies that will be packaged into an EE container. |
| `requirements.txt`          | Defines the Python dependencies that will be installed into the container.                                                                                                  |
| `requirements.yml`          | Defines the Ansible collections and roles that will be installed into the container.                                                                                        |

These files are fairly standard if you are used to working with Ansible content and containers.  To summarize what these files are used for, when you build an EE, you use a command line tool called `ansible-builder`.  This tool expects these files as part of its convention.  The tool will construct a file to build the container with all of the dependencies that have been listed in these files.

The `execution-environment.yml` is fairly unique to this EE because it performs a number of tasks to install CLI tools.  There are two primary build steps in this file `prepend` and `append`.  The `prepend` operations happen prior to Ansible attempting to install the content defined in `requirements.txt` and `requirements.yml`.  The `append` operations happen after the content installation from those files happens.  For the [Cloud EE][cloud-ee], anything installed through package managers is installed via `prepend`, while anything installed through scripts or other tools is installed via `append`.

### Building a Multi-Architecture Container

If you are building EEs for `amd64` based systems only, the running the standard commands that are documented for [Ansible Builder][ansible-builder] will work out-of-the-box.  However, if you want the containers to run on a M1 Mac, then there are a few extra steps.

Clone the [Cloud EE][cloud-ee] repository to your computer.

```bash
git clone git@github.com:scottharwell/cloud-ee.git
```

Run `ansible-builder`'s create command to generate a `Dockerfile`.  This file will be used with Docker to build multi-arch containers, but it explicitly tells Docker that the upstream container only supports `amd64` architectures.

```bash
ansible-builder create --output-filename Dockerfile; gsed -i 's/FROM/FROM --platform=linux\/amd64/' context/Dockerfile
```

Change directory into the `context` directory that was created in the previous step.

```bash
cd context
```

Use Docker's `buildx` tool to build a multi-arch image that will virtualize `amd64` on `arm64` CPUs.  You would replace `quay.io/scottharwell` with your registry and namespace.

**Note:** this step will fail if you haven't configured your Red Hat developer account and logged in to the Red Hat container registry first since the container file expects to pull the downstream containers.

```bash
docker buildx build --no-cache --platform linux/arm64 -t quay.io/scottharwell/cloud-ee:local . --push
```

The build process can take a while.  On my M1 Max, it typically takes about 45 minutes to build the container because of the `arm64` virtualization work.  Building just for `amd64` is much faster on a native CPU.

### Using the EE

Once built, you can use the EE in any Ansible tool that supports EEs such as AWX, Ansible Automation Controller, Ansible Runner, and Ansible Navigator.  For the sake of simplicity, I'll demonstrate a command using this EE to run a playbook that is in a local folder on my computer using the [AWS Demos][aws-demos] from the Ansible Cloud Content Lab.  This playbook will use a few of the AWS collections for Ansible, which were previously installed into the EE so they are always available when I need to run automation for AWS, Azure, etc.

This command uses `ansible-navigator run` in a similar manner that `ansible run` was used prior to EEs.  It basically tells `ansible-navigator` how to run the playbook, inventory information, environment variables, and extra-vars used to run the automation.  This utility is really great because I can map local folders from my Mac to pass files, such as ssh keys, to the container as automation runs.  These would be credentials if I were to run the same playbook in AWX or Ansible Automation Controller.

```bash
ansible-navigator run playbook_create_transit_network.yml \
--pae false \
--mode stdout \
--ee true \
--ce docker \
--pp always \
--eei quay.io/scottharwell/cloud-ee:latest \
--penv "AWS_ACCESS_KEY" \
--penv "AWS_SECRET_ACCESS_KEY" \
--eev $HOME/.ssh:/home/runner/.ssh \
--extra-vars "dmz_ssh_key_name=aws-test-key" \
--extra-vars "priv_network_ssh_key_name=aws-test-key" \
--extra-vars "dmz_instance_ami=ami-06640050dc3f556bb" \
--extra-vars "priv_network_instance_ami=ami-06640050dc3f556bb" \
--extra-vars "aws_region=us-east-1" \
--extra-vars "ansible_ssh_private_key_file_local_path=~/.ssh/aws-test-key.pem" \
--extra-vars "ansible_ssh_private_key_file_dest_path=~/.ssh/aws-test-key.pem" \
--extra-vars "ansible_ssh_user=ec2-user" \
--extra-vars "ansible_ssh_private_key_file=~/.ssh/aws-test-key.pem" \
--extra-vars "priv_network_ssh_user=ec2-user" \
--extra-vars "priv_network_hosts_pattern=10.*"
```

## Using the EE

I keep a new version of the [EE on quay.io][quay-cloud-ee] whenever any of the collections within the EE are incremented.  This means that new builds happen regularly.  This also means that you can use this execution environment without having to build it yourself.  But, all of the files are on [GitHub][cloud-ee] if you're interested in building and extending the EE for your own use cases.

[cloud-ee]: https://github.com/scottharwell/cloud-ee
[quay-cloud-ee]: https://quay.io/repository/scottharwell/cloud-ee
[rh-developer]: https://developers.redhat.com/blog/2016/03/31/no-cost-rhel-developer-subscription-now-available
[ansible-builder]: https://www.ansible.com/blog/introduction-to-ansible-builder
[aws-demos]: https://github.com/ansible-content-lab/aws.infrastructure_config_demos
