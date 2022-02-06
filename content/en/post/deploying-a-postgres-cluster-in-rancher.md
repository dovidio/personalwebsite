---
title: Deploying a Postgres cluster in Rancher
date: 2022-02-06T10:03:16.171Z
image: /images/uploads/1-lwidapsds7llkqxdfi_oqg.png
description: Deploying a Postgres cluster on Kubernetes has become quite easy
  thanks to the [Postgres
  Operator](https://github.com/zalando/postgres-operator). Let's see how this
  can be done with a rancher cluster.
subtitle: Deploying a Postgres cluster on Kubernetes has become quite easy
  thanks to the [Postgres
  Operator](https://github.com/zalando/postgres-operator). Let's see how this
  can be done with a rancher cluster.
---
Deploying a Postgres cluster on Kubernetes has become quite easy thanks to the [Postgres Operator](https://github.com/zalando/postgres-operator). Let's see how this can be done with a rancher cluster.

<-- more -->

Essentially, it's sufficient to follow the [quickstart tutorial](https://github.com/zalando/postgres-operator/blob/master/docs/quickstart.md). There is only a caveat. I am running a bare metal K3s cluster. By default, the operator will try to create a `PersistentVolumeClaim` with the default storage class of the Cluster. Problem is, my cluster didn't have a default storage class. The `PersistentVolumeClaim` fails, and looking at the events it becomes clear what's the problem: `FailedBinding: no persistent volumes available for this claim and no storage class is set`.

![Failed persistent volume binding](/images/uploads/screenshot-2022-02-06-at-11.37.30.png)

To solve this, I used the [Rancher Local Path Provisioner](https://github.com/rancher/local-path-provisioner) which is good enough since the cluster I'm working with is used only for testing. We first create the storage class with the following command:

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

And then we set it as default storage class with 

```bash
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

Here's a recap of all commands I used to deploy the postgres cluster:
```bash
# setup default storage class to be local-path
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
# deploy the operator
kubectl apply -k github.com/zalando/postgres-operator/manifests
# deploy a minimal cluster
kubectl create -f manifests/minimal-postgres-manifest.yaml

```