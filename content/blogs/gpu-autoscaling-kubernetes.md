---
title: "GPU-Aware Autoscaling on Kubernetes with Karpenter and KEDA"
date: 2025-03-01
draft: false
author: "Pranav Bhendawade"
tags: ["Kubernetes", "GPU", "Karpenter", "KEDA", "AI/ML", "Platform Engineering"]
description: "How I built a GPU-aware autoscaling platform on EKS that dynamically provisions nodes based on GPU utilization and job queue depth — cutting infrastructure costs by 34%."
image: /images/projects/gpu-autoscaling.svg
---

As AI/ML workloads become the new normal in platform engineering, the challenge shifts from *"can we run GPU jobs on Kubernetes?"* to *"can we do it without burning through budget and keeping latency low?"*

This post walks through how I built a GPU-aware autoscaling platform on AWS EKS using **Karpenter** for node provisioning and **KEDA** for event-driven pod scaling — achieving a 34% infrastructure cost reduction while keeping inference SLAs intact.

## The Problem

Running GPU workloads at scale has two conflicting requirements:

- **Always-on capacity** for low-latency inference (expensive)
- **Burst capacity** for batch training jobs (unpredictable)

Static node pools fail at both — you either over-provision (wasteful) or under-provision (degraded SLAs).

## The Architecture

```
┌─────────────────────────────────────────────┐
│              EKS Cluster                    │
│                                             │
│  ┌─────────┐    ┌────────┐    ┌──────────┐ │
│  │  KEDA   │───▶│  HPA   │───▶│   Pods   │ │
│  │ScaledObj│    │        │    │ (vLLM /  │ │
│  └────┬────┘    └────────┘    │ PyTorch) │ │
│       │                       └──────────┘ │
│  GPU metrics                               │
│  + queue depth                             │
│       │         ┌────────────┐             │
│       └────────▶│  Karpenter │             │
│                 │ NodePool   │             │
│                 └─────┬──────┘             │
│                       │                   │
│              AWS EC2 GPU Nodes             │
│         (p3.2xlarge / g4dn.xlarge)        │
└─────────────────────────────────────────────┘
```

### Karpenter for Node Provisioning

Karpenter watches for unschedulable pods and provisions the right node type — no more pre-warming node groups.

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: gpu-nodepool
spec:
  template:
    spec:
      requirements:
        - key: karpenter.k8s.aws/instance-gpu-count
          operator: Gt
          values: ["0"]
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: gpu-node-class
  limits:
    nvidia.com/gpu: 32
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 5m
```

The key here is `consolidateAfter: 5m` — Karpenter will deprovision idle GPU nodes after 5 minutes of underutilization, which is the primary cost-saving lever.

### KEDA for Event-Driven Scaling

KEDA scales pods based on **GPU utilization** (via Prometheus) and **job queue depth** — not just CPU/memory like standard HPA.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: gpu-inference-scaler
spec:
  scaleTargetRef:
    name: vllm-inference
  minReplicaCount: 1
  maxReplicaCount: 8
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: gpu_utilization_avg
        threshold: "70"
        query: avg(DCGM_FI_DEV_GPU_UTIL{namespace="inference"})
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: job_queue_depth
        threshold: "5"
        query: sum(inference_queue_depth)
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Infrastructure cost | Baseline | **-34%** |
| Node provisioning time | 8-12 min | **< 90s** |
| GPU utilization | ~40% | **~78%** |
| Inference P95 latency | 450ms | **390ms** |

## Key Takeaways

1. **Karpenter + KEDA is the right combo** — Karpenter handles the node layer, KEDA handles the workload layer. They're complementary, not competing.

2. **GPU consolidation is the biggest cost lever** — Most clusters keep GPU nodes warm "just in case." Setting an aggressive `consolidateAfter` on Karpenter with a fast provisioning time makes this safe.

3. **Queue depth is a better signal than utilization alone** — GPU utilization lags behind actual demand. Scaling on queue depth gives you ahead-of-time capacity.

4. **Profile your GPU instance types** — `g4dn.xlarge` (1x T4) is 4x cheaper than `p3.2xlarge` (1x V100) for inference. Match the instance to the workload.

---

*Have questions or want to discuss GPU platform engineering? Reach out at [pranavbhendawade@gmail.com](mailto:pranavbhendawade@gmail.com)*
