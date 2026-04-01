# Autopilot ComputeClasses on Standard Clusters

GKE now lets you run Autopilot workloads inside Standard clusters using ComputeClasses. You select a ComputeClass in your pod spec with a `nodeSelector`, and GKE provisions and manages the node. The rest of the cluster stays Standard.

Useful when some pods need node-level control (DaemonSets, SSH, custom node configs) and others just need to run somewhere.

## What this covers

- Using built-in Autopilot ComputeClasses (`autopilot`, `autopilot-spot`) on a Standard cluster
- Creating a custom Autopilot ComputeClass with machine family priorities
- How GKE provisions and removes nodes for ComputeClass pods
- Resource mutation — GKE bumping your requests to meet minimums
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

## Step 1 — Create a Standard cluster

You need a Standard cluster on the rapid channel with GKE version 1.34.1+. Autopilot ComputeClasses on Standard require this version. Takes about 3–5 minutes.

```bash
gcloud container clusters create $CLUSTER_NAME \
  --zone $ZONE \
  --num-nodes 1 \
  --machine-type e2-medium \
  --release-channel rapid \
  --enable-autoprovisioning \
  --max-cpu 20 \
  --max-memory 64
```

`--enable-autoprovisioning` lets GKE create node pools automatically when a ComputeClass needs them.

```bash
gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE
kubectl get nodes
```

You should see 1 node in `Ready` state.

Check that the built-in ComputeClasses are available:

```bash
kubectl get computeclasses
```

You should see `autopilot` and `autopilot-spot` in the list.

## Step 2 — Deploy with the built-in `autopilot` ComputeClass

The `autopilot` ComputeClass is built into GKE. Select it with a `nodeSelector` — no ComputeClass CR to create, no node pool to configure.

```bash
kubectl apply -f manifests/workload-autopilot.yaml
kubectl get pods -w
```

Pod stays `Pending` for 1–3 minutes while GKE creates and configures a node. Then it goes `Running`.

Check nodes:

```bash
kubectl get nodes -L cloud.google.com/compute-class
```

You should see a second node with the `cloud.google.com/compute-class=autopilot` label. GKE created and manages this node.

## Step 3 — Deploy with `autopilot-spot` ComputeClass

The `autopilot-spot` class works the same way but provisions Spot VMs. Good for batch jobs or fault-tolerant workloads.

```bash
kubectl apply -f manifests/workload-autopilot-spot.yaml
kubectl get pods -w
```

Once running, check nodes:

```bash
kubectl get nodes -L cloud.google.com/compute-class,cloud.google.com/gke-spot
```

You should see:
- `autopilot` node (on-demand)
- `autopilot-spot` node with `cloud.google.com/gke-spot=true`

See which pod is on which node:

```bash
kubectl get pods -o wide
```

## Step 4 — Create and use a custom ComputeClass

Built-in classes cover general-purpose workloads. If you need specific machine families, create a custom ComputeClass.

This one targets N4 machines, prefers Spot, and falls back to on-demand:

```bash
kubectl apply -f manifests/computeclass-n4.yaml
kubectl get computeclasses
```

Now deploy a workload that selects it:

```bash
kubectl apply -f manifests/workload-custom-class.yaml
kubectl get pods -w
```

GKE provisions an N4 node based on the ComputeClass priorities. Check:

```bash
kubectl get nodes -L cloud.google.com/compute-class
```

You should see a node with `cloud.google.com/compute-class=n4-class`.

## Step 5 — Resource mutation

Each ComputeClass has minimum resource requirements. If your pod requests less, GKE bumps the values up silently.

Check what a pod actually got:

```bash
POD=$(kubectl get pod -l app=workload-autopilot -o jsonpath='{.items[0].metadata.name}')
kubectl get pod $POD -o jsonpath='{.spec.containers[0].resources}' | python3 -m json.tool
```

Compare with `manifests/workload-autopilot.yaml`. If the numbers differ, GKE mutated them. This affects billing — you pay for the actual requests, not what you wrote in the manifest.

## Step 6 — Standard workload still works normally

Deploy a pod without a ComputeClass selector:

```bash
kubectl apply -f manifests/workload-standard.yaml
kubectl get pods -o wide
```

It schedules on the original `e2-medium` node, not on any Autopilot-managed node.

## Step 7 — Scale-down

Delete the autopilot workload:

```bash
kubectl delete -f manifests/workload-autopilot.yaml
kubectl get nodes -L cloud.google.com/compute-class -w
```

The `autopilot` node sticks around for a few minutes (cooldown to avoid thrashing), then GKE removes it.

## Step 8 — Clean up

```bash
kubectl delete -f manifests/
gcloud container clusters delete $CLUSTER_NAME --zone $ZONE --quiet
```

## Summary

Autopilot pods got their own nodes provisioned by GKE. Standard pods used the original node pool. The built-in `autopilot` and `autopilot-spot` classes worked out of the box. The custom `n4-class` let you target a specific machine family while still having GKE manage the node.