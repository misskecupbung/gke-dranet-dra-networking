# DRANET — Dynamic Resource Allocation for Network Interfaces on GKE

Standard pods get one NIC. For distributed GPU training using RDMA (GPU-to-GPU over InfiniBand/RoCE without CPU), pods need specific NICs — ideally on the same PCIe bus as the GPU.

DRA (Dynamic Resource Allocation) replaces the old device plugin model. DRANET is GKE's DRA driver for network interfaces.

## What this covers

- DRA concepts: ResourceSlice, DeviceClass, ResourceClaim
- Requesting a NIC via ResourceClaim
- Topology constraints (NIC near GPU)
- Multi-node alignment
- DRA vs device plugin comparison

## Prerequisites

- `gcloud` CLI authenticated
- `kubectl` installed
- GCP project with billing enabled
- DRA is supported on GKE 1.31+

## DRA vs device plugins

| | Device Plugin | DRA |
|---|---|---|
| Resource declaration | `resources.limits` key-value | `ResourceClaim` with rich constraints |
| Topology constraints | Not supported | Supported (CEL expressions) |
| Shared devices | Not possible | Possible |
| Cross-node alignment | Not possible | Supported |

## Set variables

```bash
export PROJECT_ID=$(gcloud config get-value project)
export CLUSTER_NAME=lab-dranet
export ZONE=us-central1-a
```

## Enable required APIs

On a new project, these APIs are not enabled by default. Run this before creating any resources.

```bash
gcloud services enable \
  container.googleapis.com \
  compute.googleapis.com \
  --project $PROJECT_ID
```

## Clone the repo

```bash
git clone https://github.com/misskecupbung/gke-dranet-dra-networking.git
cd gke-dranet-dra-networking
```

## Step 1 — Create a cluster with DRANET

The cluster needs `--enable-multi-networking` for multiple NICs per node and `--network-performance-config` with TIER_1 for high-bandwidth interfaces. DRANET runs automatically on GKE 1.31+ — no extra installation needed.

`--disk-size=50` and `--disk-type=pd-standard` keep the node boot disk small and cheap. pd-standard (HDD) is enough for this lab since we are not doing heavy disk I/O.

```bash
gcloud container clusters create $CLUSTER_NAME \
  --zone $ZONE \
  --num-nodes 2 \
  --machine-type n2-standard-8 \
  --disk-size=50 \
  --disk-type=pd-standard \
  --network-performance-config total-egress-bandwidth-tier=TIER_1 \
  --enable-multi-networking \
  --enable-dataplane-v2 \
  --release-channel rapid

gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE
```

Verify DRANET is running:

```bash
kubectl get pods -n kube-system -l component=dranet
```

## Step 2 — Inspect available devices

DRANET runs as a DaemonSet and publishes a `ResourceSlice` per node. Each slice lists the NICs on that node with their hardware attributes — bandwidth, NUMA node, PCIe location, driver.

```bash
kubectl get resourceslice
```

Look at the details of one slice:

```bash
kubectl get resourceslice \
  $(kubectl get resourceslice -o jsonpath='{.items[0].metadata.name}') \
  -o yaml | head -60
```

You will see entries like bandwidth, NUMA node, and driver name.

```bash
kubectl get deviceclass
kubectl describe deviceclass google-dranet-nic
```

## Step 3 — Create a ResourceClaim

A `ResourceClaim` is how a workload requests a device. It stays `Pending` until a pod that references it is scheduled. DRA allocates lazily at scheduling time, not at claim creation.

```bash
kubectl apply -f manifests/resource-claim.yaml
kubectl get resourceclaim basic-nic-claim
kubectl describe resourceclaim basic-nic-claim
```

## Step 4 — Deploy a pod with the NIC claim

```bash
kubectl apply -f manifests/pod-with-nic.yaml
kubectl get pods -w
```

Once the pod is running, check the claim:

```bash
kubectl describe resourceclaim basic-nic-claim
```

The claim now shows `Allocated: true` with the specific device and node.

```bash
kubectl exec pod-with-nic -- ip link show
kubectl exec pod-with-nic -- ip -details link show
```

You can see the DRANET-managed interface available in the pod's network namespace.

## Step 5 — Topology constraints

For GPU training, the NIC should be on the same NUMA node as the GPU to avoid crossing a PCIe bridge during RDMA transfers. DRA uses a CEL expression to say exactly which device attributes are required.

With the old device plugin model, you could not express this. GPU and NIC were allocated independently with no guarantee they were close.

```bash
kubectl apply -f manifests/resource-claim-topology.yaml
kubectl apply -f manifests/pod-with-topology-nic.yaml

kubectl describe resourceclaim topology-nic-claim | grep -A20 "Allocation"
```

## Step 6 — Multi-node alignment

DRA also supports allocating devices across multiple nodes for a single job. For training across 4 nodes, you can require the NIC on each node to meet the same constraints. The scheduler checks all allocations are compatible before binding any pod.

This matters for all-reduce collectives: if one node's NIC is 10 Gbps and the others are 100 Gbps, the whole job runs at 10 Gbps.

## Step 7 — See DRA during scheduling

```bash
kubectl get events --sort-by='.lastTimestamp' | grep -i "resource\|claim\|dra"
```

You see events for when the claim was allocated and when the pod was bound. The scheduler picks a node where the requested device exists — different from device plugins where resources were just counted.

## Step 8 — Clean up

```bash
kubectl delete -f manifests/ --ignore-not-found
gcloud container clusters delete $CLUSTER_NAME --zone $ZONE --quiet
```

## Resources

- [Kubernetes DRA documentation](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)
- [DRANET on GKE documentation](https://cloud.google.com/kubernetes-engine/docs/how-to/allocate-network-resources-dra)
