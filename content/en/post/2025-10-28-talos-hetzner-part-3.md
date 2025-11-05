---
title: "Kubernetes on Hetzner with Talos: Part 3 - Setting Up Longhorn Storage"
date: 2025-10-28T08:00:00+02:00
draft: false
tags: ["kubernetes", "talos", "hetzner", "longhorn", "storage"]
---

## Overview

In the [previous parts of this series](/post/2025-10-27-talos-hetzner-part-2/), we bootstrapped a Kubernetes cluster and installed Argo CD. Now it's time to add persistent storage with Longhorn. Longhorn is a lightweight, reliable, and easy-to-use distributed block storage system for Kubernetes. It's perfect for our self-hosted cluster as it provides data replication, backup capabilities, and a nice web UI for management.

If you haven't followed the previous parts, make sure you have:
- A working Talos Kubernetes cluster ([Part 1](/post/2025-10-26-talos-hetzner-part-1/))
- Argo CD installed and configured ([Part 2](/post/2025-10-27-talos-hetzner-part-2/))

## Why Longhorn?

For a self-hosted cluster, Longhorn offers several advantages:

- **Built for Kubernetes** - Native Kubernetes storage that integrates seamlessly
- **Data Resilience** - Automatic replication across nodes prevents data loss
- **Easy Management** - Web UI for monitoring and managing storage
- **Backup Support** - Built-in backup capabilities to external storage
- **No External Dependencies** - Uses local storage on your nodes

## Prerequisites

- Talos Kubernetes cluster from Part 1
- Argo CD installed from Part 2
- `talosctl` configured to access your cluster
- `kubectl` configured to access your cluster

## Step 1: Install Required Talos Extensions

Longhorn requires specific system extensions to function properly. We need to add `iscsi-tools` and `util-linux-tools` to each node.

First, let's generate a custom Talos image with the required extensions:

```bash
curl -X POST --data-binary @- https://factory.talos.dev/schematics <<EOF
customization:
  systemExtensions:
    officialExtensions:
      - siderolabs/iscsi-tools
      - siderolabs/util-linux-tools
EOF
```

This will return a schematic ID that we'll use for the upgrade:

```json
{"id":"613e1592b2da41ae5e265e8789429f22e121aab91cb4deb6bc3c0b6262961245"}
```

Save this schematic ID as an environment variable:

```bash
export SCHEMATIC_ID="613e1592b2da41ae5e265e8789429f22e121aab91cb4deb6bc3c0b6262961245"
```

## Step 2: Upgrade Each Node with Extensions

Now we need to upgrade each node to include the required extensions. We'll do this one node at a time:

```bash
# Get the list of your nodes
kubectl get nodes -o wide

# Upgrade each node (replace with your actual node IPs)
export NODE1_IP="your-node-1-ip"
export NODE2_IP="your-node-2-ip" 
export NODE3_IP="your-node-3-ip"

# Upgrade node 1
talosctl upgrade \
  --nodes $NODE1_IP \
  --image factory.talos.dev/installer/$SCHEMATIC_ID:v1.11.3

# Wait for node 1 to come back online, then upgrade node 2
talosctl upgrade \
  --nodes $NODE2_IP \
  --image factory.talos.dev/installer/$SCHEMATIC_ID:v1.11.3

# Wait for node 2 to come back online, then upgrade node 3
talosctl upgrade \
  --nodes $NODE3_IP \
  --image factory.talos.dev/installer/$SCHEMATIC_ID:v1.11.3
```

Each node will reboot during the upgrade. Wait for each node to come back online before upgrading the next one.

## Step 3: Verify Extensions Installation

After all nodes have been upgraded, verify that the extensions are installed:

```bash
# Check extensions on each node
talosctl get extensions -n $NODE1_IP
talosctl get extensions -n $NODE2_IP
talosctl get extensions -n $NODE3_IP
```

You should see `iscsi-tools` and `util-linux-tools` listed in the output.

## Step 4: Create Longhorn Namespace

Create a dedicated namespace for Longhorn with the necessary security permissions:

```bash
kubectl create namespace longhorn-system
kubectl label namespace longhorn-system pod-security.kubernetes.io/enforce=privileged
kubectl label namespace longhorn-system pod-security.kubernetes.io/audit=privileged
kubectl label namespace longhorn-system pod-security.kubernetes.io/warn=privileged
```

These labels are required because Longhorn needs privileged access to manage storage on the nodes.

## Step 5: Deploy Longhorn via Argo CD

Now let's use Argo CD to deploy Longhorn. Create a file called `longhorn-app.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: longhorn
  namespace: argocd
spec:
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
  project: default
  sources:
    - chart: longhorn
      repoURL: https://charts.longhorn.io/
      targetRevision: v1.10.0
      helm:
        values: |
          preUpgradeChecker:
            jobEnabled: false
  destination:
    server: https://kubernetes.default.svc
    namespace: longhorn-system
```

Apply the application using kubectl:

```bash
kubectl apply -f longhorn-app.yaml
```

## Step 6: Monitor the Deployment

You can monitor the deployment progress in the Argo CD dashboard or via the CLI:

```bash
# Check Argo CD application status
argocd app get longhorn

# Watch Longhorn pods starting up
kubectl get pods -n longhorn-system -w
```

The deployment might take a few minutes. You may need to trigger a sync in Argo CD if some resources appear degraded initially - this is normal as some components wait for others to be ready.

## Step 7: Access the Longhorn Dashboard

Once Longhorn is deployed, you can access its web interface using port forwarding:

```bash
kubectl port-forward svc/longhorn-frontend -n longhorn-system 8081:80
```

Then open your browser to `http://localhost:8081` to see the Longhorn dashboard. You'll see all your nodes and their available storage.

For convenience, you can create an alias:

```bash
alias longhorn_dashboard="kubectl port-forward svc/longhorn-frontend -n longhorn-system 8081:80"
```

## Step 8 (Optional): Adding External Storage

If you have additional storage volumes in Hetzner that you want Longhorn to use, here's how to set them up:

### Attach the Volume
First, attach the volume to one of your nodes via the Hetzner Cloud Console.

### Configure Talos to Mount the Volume
Create a patch file called `longhorn-storage-patch.yaml` to expose the volume as a user volume:

```yaml
# longhorn-storage-patch.yaml
machine:
  disks:
    - device: /dev/sdb  # Replace with your volume device
      partitions:
        - mountpoint: /var/mnt/longhorn-extra-storage
```

If the volume is already formatted, you might need to wipe it first using talosctl:

```bash
talosctl wipe disk sdb -n $NODE_IP
```

Then apply the patch using talosctl and verify:

```bash
talosctl machineconfig patch controlplane.yaml --patch @longhorn-storage-patch.yaml -o controlplane-updated.yaml
# Apply the updated config to the specific node
talosctl get volumestatus -n $NODE_IP
```

### Add Disk in Longhorn UI
Once the volume is mounted, go to the Longhorn dashboard, navigate to "Node" -> select your node -> "Edit Disks" -> "Add Disk" and add the new storage path.

## What's Next?

Now that we have persistent storage set up, we're ready to deploy stateful applications! In the next part of this series, we'll deploy Navidrome for music streaming, which will demonstrate how to use Longhorn storage for real applications.

## Summary

We've successfully installed Longhorn distributed storage on our Talos Kubernetes cluster. Our setup now includes:

- Talos nodes with required storage extensions
- Longhorn deployed via Argo CD for GitOps management  
- A web dashboard for storage monitoring
- Persistent storage ready for stateful applications

With Longhorn in place, we now have a production-ready storage solution that can handle data replication, backups, and recovery for our self-hosted applications.