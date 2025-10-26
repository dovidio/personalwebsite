---
title: "Kubernetes on Hetzner with Talos: Part 1 - Bootstrapping Control Plane Nodes"
date: 2025-10-26T08:00:00+02:00
draft: false
tags: ["kubernetes", "talos", "hetzner"]
---

## Overview

This is the first part of a multi-part series on setting up a Kubernetes cluster on Hetzner Cloud using Talos OS. In this article, we'll focus on bootstrapping 3 control plane nodes that can also run workloads.

My motivations for this project are threefold:
1. **Learning** - I wanted to understand Kubernetes internals by building a cluster from scratch
2. **Self-hosting** - Having a place to run apps I use daily (like [Navidrome](https://www.navidrome.org/) for music)
3. **Side projects** - A cluster to deploy and experiment with my own projects

If you don't have similar requirements, this series probably isn't worth your time. But if you're a hobbyist looking to learn and host your own services for cheap, read on.
I don't administer Kubernetes clusters for a leaving - this is a hobby project. If you're looking for enterprise-grade guidance from a professional Kubernetes administrator, this might not be the right series for you.
You might also want to read [Mathias Pius' excellent series](https://datavirke.dk/posts/bare-metal-kubernetes-part-1-talos-on-hetzner/) that I frequently consulted whenever I got stuck, as well as the official Talos Hetzner [tutorial](https://docs.siderolabs.com/talos/v1.10/platform-specific-installations/cloud-platforms/hetzner).

## Why Talos + Hetzner?

Talos OS is a minimal, hardened Linux distribution designed specifically for Kubernetes. Combined with Hetzner's affordable cloud infrastructure, you get a secure, manageable cluster at a fraction of the cost of major cloud providers.
To further keep costs down, we'll use a three node setup, where each node is running the control plane and the actual workloads. The total costs will be less than 20 euro per month.
The trade-off is slightly less isolation between control plane and workloads, but for many use cases, this is perfectly acceptable.

## Prerequisites

- Hetzner Cloud account and API token with read/write permissions. Set the `HCLOUD_TOKEN` environment variable with the API token. Additionally, make sure to have Hetzner's [hcloud tool](https://github.com/hetznercloud/cli) installed
- [talosctl](https://docs.siderolabs.com/talos/v1.11/getting-started/talosctl) installed
- Basic understanding of Kubernetes concepts. For absolute newbies you can still follow along, but I suggest supplementing this tutorial with the excellent book [Kubernetes Up & Running](https://www.amazon.com/Kubernetes-Running-Dive-Future-Infrastructure/dp/1492046531)
- [packer](https://developer.hashicorp.com/packer/install) to create the initial image snapshot.
- [helm](https://helm.sh/) to install Hetzner cloud control manager.

## Step 1: Create Talos Image with Packer

First, we need a custom Talos image that Hetzner can use. We'll use Packer to build and upload this image.

```hcl
packer {
  required_plugins {
    hcloud = {
      source  = "github.com/hetznercloud/hcloud"
      version = "~> 1"
    }
  }
  }

  variable "talos_version" {
  type    = string
  default = "v1.11.3"
  }

  variable "arch" {
  type    = string
  default = "amd64"
  }

  variable "server_type" {
  type    = string
  default = "cx23"
  }

  variable "server_location" {
  type    = string
  default = "nbg1"
  }

  locals {
  image = "https://factory.talos.dev/image/376567988ad370138ad8b2698212367b8edcb69b5fd68c80be1f2ec7d603b4ba/${var.talos_version}/hcloud-${var.arch}.raw.xz"
  }

  source "hcloud" "talos" {
  rescue       = "linux64"
  image        = "debian-11"
  location     = "${var.server_location}"
  server_type  = "${var.server_type}"
  ssh_username = "root"

  snapshot_name   = "talos system disk - ${var.arch} - ${var.talos_version}"
  snapshot_labels = {
    type    = "infra",
    os      = "talos",
    version = "${var.talos_version}",
    arch    = "${var.arch}",
  }
  }

  build {
  sources = ["source.hcloud.talos"]

  provisioner "shell" {
    inline = [
      "apt-get install -y wget",
      "wget -O /tmp/talos.raw.xz ${local.image}",
      "xz -d -c /tmp/talos.raw.xz | dd of=/dev/sda && sync",
    ]
  }
  }
```

Build the image:
```bash
packer build hcloud.pkr.hcl
```

This command will do a couple of things:
1. Spin up a Debian server in Hetzner
2. Enable rescue mode and reboot the server
3. Download the Talos image from the Talos factory to override the Debian image
4. Shut down the server and create the snapshot
5. Upload the snapshot to Hetzner

The whole process took 7 minutes for me, so feel free to take a break while the automation does its magic.
Once this is done, keep track of the snapshot id, we'll use it in the following steps.
```bash
export SNAPSHOT_ID=<your snapshot id>
```

## Step 2: Set Up Load Balancer
We are going to use a load balancer for the Kubernetes API endpoint, as well as for ingress.
This is not strictly necessary at the beginning and you could save $5 per month by not using it,
but I decided to use it since I will need it anyway for my personal projects to be highly available. 

```bash
# Create load balancer
hcloud load-balancer create \
  --name talos-control-plane \
  --network-zone eu-central \
  --type lb11 \
  --label 'type=controlplane'

# Add service for Kubernetes API
hcloud load-balancer add-service \
  --name talos-control-plane \
  --protocol tcp \
  --listen-port 6443 \
  --destination-port 6443

# Add targets using labels
hcloud load-balancer add-target talos-control-plane --label-selector 'type=controlplane'
```
We first create the load balancer. After that we expose port `6443` and use the same port as destination. We create a target group using label selector, so that every time we spin up a new node with that label, it will automatically be added to the group.

Now we can find the IP of our newly created load balancer with the following command
```bash
hcloud loadbalancer list
```

Let's add this to an env variable, we are going to need it in the following steps
```bash
export LB_IP=<your load balancer ip>
```

## Step 3: Generate Talos Configuration

We are now ready to generate the talos configuration, which we'll use to connect to the cluster and to spin up new nodes.

```bash
# Generate cluster configuration
talosctl gen config talos-k8s-hcloud-tutorial https://$LB_IP:6443 \
  --with-examples=false \
  --with-docs=false
```

This creates several files, including `controlplane.yaml`, `worker.yaml`, and `talosconfig`.

## Step 4: Patch Configuration for Hetzner Integration

We are going to apply a few changes to the configuration. To do so, we'll use patches.
Let's create a patches folder where we can keep track of all patches.
We'll start by patching the cluster to allow externalCloudProviders. This will be required to use the [hetzner cloud controller manager](https://github.com/hetznercloud/hcloud-cloud-controller-manager).


```yaml
# patches/external_cloud_providers.yaml
cluster:
  externalCloudProvider:
    enabled: true
```

Next, we'll create a patch to allow workloads to be scheduled on control plane nodes.
This will prevent nodes from being tainted, which would stop workloads from scheduling.
```yaml
# patches/allow_controlplane_workloads.yaml
cluster:
  allowSchedulingOnControlPlanes: true
```

Apply the patch:
```bash
talosctl machineconfig patch controlplane.yaml --patch @patches/allow_controlplane_workloads.yaml --patch @patches/external_cloud_providers.yaml -o controlplane.yaml
```

## Step 5: Deploy Control Plane Nodes

Now we create our 3 control plane nodes using the patched configuration:

```bash
# Create three control plane nodes
for i in 1 2 3; do
  hcloud server create \
    --name talos-control-plane-$i \
    --image $SNAPSHOT_ID \
    --type cx23 \
    --location nbg1 \
    --label 'type=controlplane' \
    --user-data-from-file controlplane.yaml
done
```

For my cluster I'm using the smallest instances available as they are more than enough for my needs.

## Step 6: Bootstrap the Cluster
Let's find the first node IP using `hcloud server list | grep talos-control-plane` and export it as an environment variable:

```bash
export FIRST_NODE_IP=<your control-plane-node-ip>
```

Now let's configure the Talos client to connect to this node by setting both the endpoint and node:

```bash
talosctl --talosconfig talosconfig config endpoint $FIRST_NODE_IP 
talosctl --talosconfig talosconfig config node $FIRST_NODE_IP
```

We are ready to bootstrap [etcd](https://etcd.io) on our first control plane node

```bash
talosctl --talosconfig talosconfig bootstrap
```

You should then be able to see all three nodes with the following command:
```bash
talosctl --talosconfig talosconfig get members 
```

We can then retrieve kubeconfig and use kubectl with it

```bash
talosctl --talosconfig talosconfig kubeconfig .
export KUBECONFIG=./kubeconfig
kubectl get nodes -o wide
```



Check that everything is working:

```bash
# Check node status
kubectl get nodes -o wide
```

We can get a glimpse of cluster health using talosctl dashboard command

```bash
talosctl --talosconfig talosconfig dashboard
```

## Step 7: Hetzner Cloud Controller Manager

Setting up Hetzner cloud control manager was quite seamless, I've just followed its official [quickstart.md](https://github.com/hetznercloud/hcloud-cloud-controller-manager/blob/main/docs/guides/quickstart.md).

Create a secret containing your Hetzner API token:
```bash
kubectl -n kube-system create secret generic hcloud --from-literal=token=<hcloud API token>
```

Add the helm repository
```bash
helm repo add hcloud https://charts.hetzner.cloud
helm repo update hcloud
```

Install the chart
```bash
helm install hccm hcloud/hcloud-cloud-controller-manager -n kube-system
```

You should be able to see hcloud-cloud-controller-manager in the deployments
```bash
kubectl get deployments -n kube-system
```

## What's Next?

In the next parts of this series, we'll cover:
- Setting up ArgoCD
- Setting up storage with Longhorn
- Deploying [Navidrome](https://www.navidrome.org/) for music streaming

## Summary

We've successfully bootstrapped a 3-node Kubernetes cluster using Talos on Hetzner Cloud. Our control plane nodes can also run workloads, giving us a cost-effective, highly available setup.
