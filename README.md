# Autopilot Compute Classes on Standard Clusters

Autopilot and Standard used to be separate cluster modes — you picked one at creation and couldn't switch. Now you can use Autopilot compute classes on Standard clusters, per pod. The rest of the cluster stays Standard.

Useful when some pods need node-level control (DaemonSets, SSH, custom node configs) and others just need to run somewhere.

## What this covers

- Deploying pods with Autopilot compute classes on a Standard cluster
- How GKE provisions and removes nodes for those pods
- Resource mutation — GKE bumping your requests to meet compute class minimums
- Running Autopilot and Standard pods side by side

## Prerequisites

- `gcloud` CLI authenticated
- `kubectl` installed
- GCP project with billing enabled
- GKE API enabled:

```bash
gcloud services enable container.googleapis.com
```

## Set variables

```bash
export PROJECT_ID=$(gcloud config get-value project)
export CLUSTER_NAME=lab-autopilot-standard
export ZONE=us-central1-a
```

## Clone the repo

```bash
git clone https://github.com/misskecupbung/gke-autopilot-standard-clusters.git
cd gke-autopilot-standard-clusters
```

---

## Step 1 — Create a Standard cluster

You need a Standard cluster on the rapid channel — this feature is rapid-only for now. Takes about 3–5 minutes.

```bash
gcloud container clusters create $CLUSTER_NAME \
  --zone $ZONE \
  --num-nodes 1 \
  --machine-type e2-medium \
  --release-channel rapid
```

```bash
gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE
kubectl get nodes
```

You should see 1 node in `Ready` state.

## Step 2 — Deploy with `balanced` compute class

Adding `cloud.google.com/compute-class` annotation to a pod tells GKE to provision a node for it automatically. No node pool setup needed.

| Compute class    | What it does                          |
|------------------|---------------------------------------|
| `general-purpose` | Default CPU/memory ratio             |
| `balanced`        | Cost-optimized instances             |
| `scale-out`       | Spot VMs, for batch/tolerant jobs    |
| `performance`     | High-CPU                             |
| `accelerator`     | GPU/TPU                              |

Pods without the annotation go to your existing Standard node pool as usual.

```bash
kubectl apply -f manifests/workload-balanced.yaml
kubectl get pods -w
```

Pod stays `Pending` for 1–3 minutes while GKE creates a node. Then it goes `Running`.

Check nodes:

```bash
kubectl get nodes -L cloud.google.com/compute-class
```

You should see a second node with `cloud.google.com/compute-class=balanced`.

## Step 3 — Deploy other compute classes

```bash
kubectl apply -f manifests/workload-general.yaml
kubectl apply -f manifests/workload-scaleout.yaml
kubectl get pods -w
```

Once everything is running, check nodes:

```bash
kubectl get nodes -L cloud.google.com/compute-class,cloud.google.com/gke-spot
```

You should see:
- `balanced` node
- `general-purpose` node
- Spot node for `scale-out` (label `cloud.google.com/gke-spot=true`)

See which pod is on which node:

```bash
kubectl get pods -o wide
```

## Step 4 — Resource mutation

Each compute class has minimum resource requirements. If your pod requests less, GKE silently bumps the values.

Check what a pod actually got:

```bash
POD=$(kubectl get pod -l app=workload-balanced -o jsonpath='{.items[0].metadata.name}')
kubectl get pod $POD -o jsonpath='{.spec.containers[0].resources}' | python3 -m json.tool
```

Compare with `manifests/workload-balanced.yaml`. If the numbers differ, GKE mutated them. This affects billing — you pay for the actual requests, not what you wrote in the manifest.

## Step 5 — Standard workload still works normally

Deploy a pod without a compute class annotation:

```bash
kubectl apply -f manifests/workload-standard.yaml
kubectl get pods -o wide
```

It schedules on the original `e2-medium` node, not on any Autopilot-managed node.

## Step 6 — Scale-down

Delete the balanced workload:

```bash
kubectl delete -f manifests/workload-balanced.yaml
kubectl get nodes -L cloud.google.com/compute-class -w
```

The `balanced` node sticks around for ~10 minutes (cooldown to avoid thrashing), then GKE removes it.

## Step 7 — Clean up

```bash
kubectl delete -f manifests/
gcloud container clusters delete $CLUSTER_NAME --zone $ZONE --quiet
```

## Summary

Autopilot pods got their own nodes provisioned by GKE. Standard pods used the original node pool. No extra node pool configuration was needed for the Autopilot workloads.

## Try next

- `accelerator` compute class for a quick GPU test without setting up a GPU node pool
- Deploy DaemonSets and see that they only land on Standard nodes
- Check compute class resource limits in [the docs](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-compute-classes)
