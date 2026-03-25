# k8s-mc

Basic Kubernetes setup for a Minecraft Java server.

This repo is intentionally small. It gives you a clean starting point that you can apply to a local cluster, a home lab, or a cloud Kubernetes cluster without a lot of extra moving parts.

## What is in here

- `k8s/namespace.yaml`
- `k8s/pvc.yaml`
- `k8s/service.yaml`
- `k8s/statefulset.yaml`
- `k8s/kustomization.yaml`

The server image used here is [`itzg/minecraft-server`](https://github.com/itzg/docker-minecraft-server). It is the most common choice for running Minecraft on Kubernetes because it already handles EULA acceptance, server setup, and world data storage cleanly.

## Prerequisites

Before you deploy anything, make sure you have:

- A working Kubernetes cluster
- `kubectl` pointed at the right cluster
- A default `StorageClass` in the cluster, or a plan to edit the PVC file to match your storage setup
- A way to reach one Kubernetes node from your network

You can check the basics with:

```bash
kubectl config current-context
kubectl get nodes
kubectl get storageclass
```

If `kubectl get storageclass` shows one marked as `(default)`, the included PVC should work without changes.

## Repo layout

```text
k8s/
  kustomization.yaml
  namespace.yaml
  pvc.yaml
  service.yaml
  statefulset.yaml
```

## Step 1: Review storage size

Open [`k8s/pvc.yaml`](/Users/billy/Documents/projects/k8s-mc/k8s-mc/k8s/pvc.yaml) and check the requested disk size.

The default here is `10Gi`, which is fine for a small personal server. If you expect large worlds, lots of exploration, or many backups, increase it now before players start using the server.

## Step 2: How players will connect

The included service is a `NodePort`.

That keeps the setup basic and avoids depending on a cloud load balancer. Players connect to the IP address of one Kubernetes node and the node port exposed by the service.

## Step 3: Deploy everything

From the repo root:

```bash
kubectl apply -k k8s
```

That will create:

- A namespace called `minecraft`
- A persistent volume claim for world data
- A service exposing Minecraft traffic
- A `StatefulSet` running one Minecraft server pod

## Step 4: Wait for the pod to come up

Use these commands:

```bash
kubectl get pods -n minecraft
kubectl get svc -n minecraft
kubectl logs -n minecraft statefulset/minecraft
```

You want to see the pod become `Running`, and the logs should show the server finishing startup.

## Step 5: Get the server address

Run:

```bash
kubectl get svc minecraft -n minecraft
kubectl get nodes -o wide
```

Connect to:

```text
<node-ip>:32565
```

## Step 6: Connect from Minecraft

From Minecraft Java Edition, add a multiplayer server using:

```text
<node-ip>:32565
```

## Useful commands

Restart the server pod:

```bash
kubectl rollout restart statefulset/minecraft -n minecraft
```

Watch logs live:

```bash
kubectl logs -f -n minecraft statefulset/minecraft
```

See the persistent volume claim:

```bash
kubectl get pvc -n minecraft
```

Delete the deployment but keep the repo files:

```bash
kubectl delete -k k8s
```

Important note: deleting the Kubernetes objects may or may not delete the underlying storage depending on your cluster's storage reclaim policy. Check that before assuming your world is gone or preserved.

## Updating the server

The manifest uses `imagePullPolicy: IfNotPresent`, which is fine for a basic setup.

If you want to refresh to a newer image version later:

```bash
kubectl delete pod minecraft-0 -n minecraft
```

Or set a pinned image tag in [`k8s/statefulset.yaml`](/Users/billy/Documents/projects/k8s-mc/k8s-mc/k8s/statefulset.yaml) and re-apply:

```bash
kubectl apply -k k8s
```

## Backups

This starter repo does not include a backup job on purpose. Keep the first version simple. If this server matters, back up the PVC or the underlying volume snapshots before you treat it like real production data.

## Troubleshooting

If the pod is stuck in `Pending`:

- Your cluster probably cannot satisfy the PVC request
- Check `kubectl describe pod minecraft-0 -n minecraft`
- Check `kubectl describe pvc minecraft-data -n minecraft`

If the pod starts and then crashes:

- Check `kubectl logs -n minecraft statefulset/minecraft`
- Make sure the node has enough memory

If nobody can connect:

- Check the node IP and node port
- Check firewall rules and home router port forwarding if you are exposing a home lab cluster
- Confirm TCP `32565` is reachable from outside the cluster

## Why a StatefulSet instead of a Deployment?

Minecraft stores state on disk and benefits from a stable pod identity. A `Deployment` can work for some cases, but a `StatefulSet` is the more natural fit when you want persistent data attached to a single long-lived server instance.
