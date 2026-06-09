---
title: "Taints and Tolerations: controlling where your pods run"
date: 2024-07-08T10:00:00+02:00
description: "The 3 effects of Kubernetes Taints (NoSchedule, PreferNoSchedule, NoExecute), their real-world use cases, and how to combine them with Tolerations."
draft: false
tags:
  - kubernetes
  - k8s
  - scheduling
  - devops
cover:
  image: cover.jpg
  alt: "Taints & Tolerations"
  relative: true
---

Taints and Tolerations let you **repel** pods from certain nodes. It is the inverse mechanism of Node Affinity (which attracts pods). The two are complementary and are often both necessary for a production workload.

## The concept

A **Taint** is placed on a **node**: it signals that the node does not accept pods by default. A **Toleration** is declared in a **pod**: it allows the pod to tolerate a specific Taint and be scheduled on that node.

Example — adding and removing a taint:

```bash
# Apply a Taint to a node
kubectl taint nodes node1 gpu=true:NoSchedule

# Remove it (trailing dash is mandatory)
kubectl taint nodes node1 gpu=true:NoSchedule-
```

## The 3 effects

A Taint has 3 effect types, each with its own use case:

### `NoSchedule`

The scheduler places no new pod on this node unless it tolerates the Taint. Pods already running on the node are **not affected**.

Use case: costly GPU nodes, nodes reserved for a specific workload.

### `PreferNoSchedule`

The scheduler **tries** to avoid this node, but will place pods there if no other option exists.

> This is a soft constraint.

Use case: nodes undergoing progressive maintenance, nodes with reduced network capacity.

### `NoExecute`

The scheduler refuses new pods **and evicts existing pods** that do not tolerate the Taint. A `tolerationSeconds` value can delay eviction.

Use case: cleanly draining a node (`kubectl drain`), incident on a node.

## Declaring a Toleration in a pod

```yaml
spec:
  tolerations:
    - key: "gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
```

The `Exists` operator tolerates the Taint regardless of its value:

```yaml
tolerations:
  - key: "gpu"
    operator: "Exists"
    effect: "NoSchedule"
```

To tolerate all Taints on a node (avoid this except in very specific cases):

```yaml
tolerations:
  - operator: "Exists"
```

### Summary

| Effect             | New pods           | Existing pods  | Constraint              |
| ------------------ | ------------------ | -------------- | ----------------------- |
| `NoSchedule`       | Blocked            | Not affected   | Strict                  |
| `PreferNoSchedule` | Avoided if possible| Not affected   | Soft                    |
| `NoExecute`        | Blocked            | Evicted        | Strict + retroactive    |

## Concrete use cases

### GPU nodes

```bash
kubectl taint nodes gpu-node-1 nvidia.com/gpu=true:NoSchedule
```

Only pods declaring the matching Toleration will be scheduled on this node. Ordinary pods will never land there, avoiding waste of an expensive resource.

<!-- SCREENSHOT: Azure portal or kubectl - list of AKS nodes with their pool (system pool vs GPU pool), showing available nodes -->

### Node maintenance

When a node must be gradually removed from service:

```bash
# Prevent new pods
kubectl taint nodes node1 maintenance=true:NoSchedule

# Then drain the node (evicts existing pods)
kubectl drain node1 --ignore-daemonsets --delete-emptydir-data
```

### `NoExecute` with eviction delay

Useful when you want to give pods time to finish their in-flight work:

```yaml
tolerations:
  - key: "maintenance"
    operator: "Equal"
    value: "true"
    effect: "NoExecute"
    tolerationSeconds: 300 # The pod has 5 minutes before being evicted
```

## Taints automatically applied by Kubernetes

AKS and Kubernetes automatically place several Taints on nodes in certain situations:

| Taint                                | Cause                    |
| ------------------------------------ | ------------------------ |
| `node.kubernetes.io/not-ready`       | Node not yet ready       |
| `node.kubernetes.io/unreachable`     | Node unreachable         |
| `node.kubernetes.io/memory-pressure` | Memory pressure          |
| `node.kubernetes.io/disk-pressure`   | Disk pressure            |

DaemonSets generally tolerate these Taints by default so they keep running even on struggling nodes.

## Taint vs Node Affinity

| Mechanism          | Side       | Direction  | Effect                                    |
| ------------------ | ---------- | ---------- | ----------------------------------------- |
| Taint + Toleration | Node + Pod | Repulsive  | Pushes away pods without a Toleration     |
| Node Affinity      | Pod        | Attractive | Pulls pods toward specific nodes          |

The two can be combined: a Taint to reserve a node, Node Affinity to attract the pods that need it toward that specific node.

## Conclusion

Taints and Tolerations are simple in appearance but very powerful in practice. They allow you to segment a cluster without multiplying clusters: reserved GPU nodes, isolated maintenance nodes, critical workloads protected from noisy neighbours. This can also be used to separate different technical tiers (e.g. frontend/backend) and to guarantee dedicated capacity when needed.

The key takeaway: a Taint without a matching Toleration blocks — and `NoExecute` is the only effect that acts on already-running pods. Choose the effect based on whether you want to protect a node going forward or drain it cleanly.

In production, Taints are almost always combined with Node Affinity: Taints prevent unwanted pods from landing on a node, Node Affinity attracts the right pods toward that same node. One repels, the other attracts. Together they give precise placement control without ambiguity and allow you to structure pod placement according to your business and technical constraints.

## Link

- [Taints and Tolerations — Kubernetes Documentation](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
