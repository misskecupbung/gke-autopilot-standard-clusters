# Autopilot ComputeClasses on Standard Clusters

GKE now lets you run Autopilot-managed workloads inside a Standard cluster using ComputeClasses. You pick a ComputeClass in your pod spec with a `nodeSelector`, and GKE creates and manages the node for that pod. The rest of the cluster stays Standard.

This is useful when some pods need node-level control (DaemonSets, SSH, custom configs) and others just need to run.

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

This needs GKE version 1.34.1 or higher on the rapid channel. That's the minimum for Autopilot ComputeClasses on Standard. Creation takes about 3–5 minutes.

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

`--enable-autoprovisioning` lets GKE create node pools on its own when a ComputeClass needs them. Without this, ComputeClass pods would stay Pending.

Once the cluster is ready, get credentials and check the node:

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

The only difference from a normal deployment is the `nodeSelector`. No node pool to create, no autoscaling to configure.

```bash
kubectl apply -f manifests/workload-autopilot.yaml
kubectl get pods -w
```

The pod stays `Pending` for 1–3 minutes while GKE creates and configures a node. Then it goes `Running`.

Check the nodes:

```bash
kubectl get nodes -L cloud.google.com/compute-class
```

You should see a second node with `cloud.google.com/compute-class=autopilot`. GKE created and manages this node.

## Step 3 — Deploy with `autopilot-spot` ComputeClass

Same as Step 2, but the node is a Spot VM. Cheaper, but GCP can stop it at any time. Good for batch jobs or workloads that can restart.

```bash
kubectl apply -f manifests/workload-autopilot-spot.yaml
kubectl get pods -w
```

Once running, check nodes:

```bash
kubectl get nodes -L cloud.google.com/compute-class,cloud.google.com/gke-spot
```

You should see an `autopilot-spot` node with `cloud.google.com/gke-spot=true`.

See which pod is on which node:

```bash
kubectl get pods -o wide
```

## Step 4 — Create and use a custom ComputeClass

The built-in classes work for general use. If you need a specific machine family, create your own ComputeClass. This one targets N4 machines — Spot first, on-demand as fallback:

```bash
kubectl apply -f manifests/computeclass-n4.yaml
kubectl get computeclasses
```

Now deploy a workload that selects it:

```bash
kubectl apply -f manifests/workload-custom-class.yaml
kubectl get pods -w
```

GKE provisions an N4 node based on the ComputeClass priorities. Once the pod is running, check the nodes:

```bash
kubectl get nodes -L cloud.google.com/compute-class
```

You should see a node with `cloud.google.com/compute-class=n4-class`.

## Step 5 — Resource mutation

Each ComputeClass has minimum resource requirements. If your pod requests less than the minimum, GKE changes the values before scheduling. It doesn't warn you.

Check what the pod actually got:

```bash
POD=$(kubectl get pod -l app=workload-custom-class -o jsonpath='{.items[0].metadata.name}')
kubectl get pod $POD -o jsonpath='{.spec.containers[0].resources}' | python3 -m json.tool
```

Compare with `manifests/workload-custom-class.yaml`. If the numbers are different, GKE changed them. This affects billing — you pay for the values in the pod spec, not what you wrote in the manifest.

## Step 6 — Standard workload still works normally

To confirm the two modes don't interfere, deploy a pod without a ComputeClass selector:

```bash
kubectl apply -f manifests/workload-standard.yaml
kubectl get pods -o wide
```

It goes to the original `e2-medium` node. The Autopilot-managed nodes are only used by pods that select them.

## Step 7 — Scale-down

Delete the custom class workload and watch the node go away:

```bash
kubectl delete -f manifests/workload-custom-class.yaml
kubectl get nodes -L cloud.google.com/compute-class -w
```

The `n4-class` node stays for a few minutes — GKE waits before removing it to avoid creating and deleting nodes too fast. After that, it's removed.

## Step 8 — Clean up

```bash
kubectl delete -f manifests/
gcloud container clusters delete $CLUSTER_NAME --zone $ZONE --quiet
```

## Summary

The built-in classes worked without any setup — just add the `nodeSelector` and GKE handles the node. The custom `n4-class` shows how to target a specific machine family while keeping the same managed behavior. Standard pods were not affected; they stayed on the original node pool.