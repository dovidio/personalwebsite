---
title: "Kubernetes on Hetzner with Talos: Part 2 - Installing Argo CD"
date: 2025-10-27T08:00:00+02:00
draft: false
tags: ["kubernetes", "talos", "hetzner", "argocd"]
---

## Overview

In the [previous part of this series](/post/2025-10-26-talos-hetzner-part-1/), we bootstrapped a 3-node Kubernetes cluster on Hetzner Cloud using Talos OS. Now we'll install Argo CD, a declarative GitOps continuous delivery tool for Kubernetes. Argo CD allows us to manage our applications using configuration as code, which means we can version control our deployments and have a clear audit trail of what's running in our cluster.

If you haven't followed Part 1 yet, go back and set up your cluster first. This tutorial assumes you have a working Kubernetes cluster and kubectl configured to connect to it.

## Why Argo CD?

For a self-hosted cluster running personal projects and side applications, Argo CD provides several benefits:

- **GitOps workflow** - Your Git repository becomes the single source of truth for your cluster state
- **Easy rollbacks** - If something breaks, you can quickly revert to a previous working state
- **Visibility** - The web UI gives you a clear view of all your applications and their sync status
- **Automation** - Argo CD can automatically sync your applications when you push changes to Git

## Prerequisites

- A working Kubernetes cluster (from Part 1)
- kubectl configured to access your cluster
- Homebrew (for installing the Argo CD CLI on macOS)

## Step 1: Create Argo CD Namespace

First, let's create a dedicated namespace for Argo CD:

```bash
kubectl create namespace argocd
```

Keeping Argo CD in its own namespace helps with organization and makes it easier to manage resources.

## Step 2: Install Argo CD in High Availability Mode

We'll install Argo CD using the official high availability manifest. Since we have a 3-node cluster, running Argo CD in HA mode makes sense for reliability:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/ha/install.yaml
```

This will deploy all the necessary components including the API server, repo server, application controller, and Redis for caching. The installation might take a minute or two as it pulls images and starts the pods.

You can watch the pods come up with:

```bash
kubectl get pods -n argocd -w
```

Wait until all pods are in the `Running` state before proceeding.

## Step 3: Install the Argo CD CLI

The Argo CD CLI makes it easy to interact with Argo CD from your terminal. Install it using Homebrew:

```bash
brew install argocd
```

## Step 4: Retrieve Initial Admin Password

Argo CD creates an initial admin password during installation. Retrieve it using the CLI:

```bash
argocd admin initial-password -n argocd
```

Alternatively, you can get it directly from the secret:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Keep this password handy - you'll need it in the next step.

## Step 5: Access the Argo CD Dashboard

The easiest way to access the Argo CD web interface is using the admin dashboard command:

```bash
argocd admin dashboard -n argocd
```

This command will automatically handle port forwarding and open your browser to the Argo CD UI. You'll see a certificate warning since we're using a self-signed certificate - that's expected for now.

You can login with:
- Username: `admin`
- Password: the password from Step 4

## Step 6: Login via CLI

Now let's login to Argo CD from the CLI:

```bash
argocd login localhost:8080
```

When prompted, use:
- Username: `admin`
- Password: the password from Step 5

You might see a warning about the certificate - type `y` to proceed.

## Step 7: Change the Admin Password

For security, change the default admin password to something you'll remember:

```bash
argocd account update-password
```

Follow the prompts to set a new password.

## Step 8: Test with the Guestbook Application

Let's deploy a test application to verify everything is working. First, switch to the argocd namespace by default to save some typing:

```bash
kubectl config set-context --current --namespace=argocd
```

Now create the guestbook application:

```bash
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

This tells Argo CD to track the guestbook application from the example repository and deploy it to the default namespace.

Sync the application to actually deploy it:

```bash
argocd app sync guestbook
```

You should see output showing Argo CD deploying the guestbook resources. Check the status:

```bash
argocd app get guestbook
```

You can also view the application in the web UI at `https://localhost:8080` - you'll see a nice visualization of all the Kubernetes resources that make up the guestbook app.

## Step 9: Accessing the UI Without Port Forwarding

The port-forward method works well for temporary access, but it requires keeping a terminal window open. For regular use, you can access the Argo CD dashboard using:

```bash
argocd admin dashboard -n argocd
```

This command will automatically open your browser and handle the port forwarding for you.

If you want to expose Argo CD to the outside world permanently, the official Argo CD documentation has detailed guides on setting up ingress with various ingress controllers. For my personal use case, I don't need external access since I can use the admin dashboard command whenever I need to check on my deployments.

## What's Next?

Now that we have Argo CD running, we have a solid foundation for managing applications in our cluster. In the next parts of this series, we'll cover:

- Setting up storage with Longhorn
- Deploying real applications like Navidrome for music streaming
- Managing those applications with Argo CD

## Summary

We've successfully installed Argo CD in high availability mode on our Talos Kubernetes cluster. We verified the installation by deploying the guestbook test application and can now access the web UI to visualize our deployments. With Argo CD in place, we're ready to start managing our applications using GitOps principles.