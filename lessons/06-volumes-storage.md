# Lesson 06 — Volumes & Persistent Storage ✅

## The problem
A container's filesystem is **ephemeral**. When a Pod is deleted/replaced, everything written
inside its container is gone (proved it: wrote `/tmp/note.txt`, deleted the Pod, the new Pod's
`cat` failed — fresh container = empty filesystem). Real apps (databases, uploads) need storage
that outlives the Pod.

## The three roles
- **PersistentVolume (PV)** — an actual piece of storage in the cluster (a real disk/dir).
- **PersistentVolumeClaim (PVC)** — a Pod's *request* for storage ("I need 100Mi, RWO"). The Pod
  mounts the **claim**, not the disk.
- **StorageClass** — the factory that **dynamically provisions** a PV to satisfy a PVC. minikube
  ships a default one (`standard`) via its storage-provisioner addon.

```
Pod  ->  PVC (a request)  ->  StorageClass (provisions)  ->  PV (the real disk)
```
Analogy: PVC = "I need a room with a king bed"; StorageClass = front desk that prepares a room;
PV = the actual room. You hold a keycard (mount), not the building.

## What I did (data survives a Pod's death)
```bash
kubectl apply -f manifests/06-storage/data-pvc.yaml
kubectl get pvc          # STATUS: Bound (matched to an auto-provisioned PV)
kubectl get pv           # a PV pvc-xxxx was auto-created by the 'standard' StorageClass
kubectl apply -f manifests/06-storage/storage-demo-pod.yaml
kubectl exec storage-demo -- sh -c 'echo "I will survive" > /data/note.txt'
kubectl delete pod storage-demo
kubectl apply -f manifests/06-storage/storage-demo-pod.yaml
kubectl exec storage-demo -- cat /data/note.txt   # STILL "I will survive" — data outlived the Pod
```
Why it survived: the file lived on the **PV (external storage)**, not the container filesystem.
The new Pod re-mounted the same PVC and found the data.

Manifests: [data-pvc.yaml](../manifests/06-storage/data-pvc.yaml) ·
[storage-demo-pod.yaml](../manifests/06-storage/storage-demo-pod.yaml)

## Access modes
| Mode | Meaning |
|------|---------|
| **ReadWriteOnce (RWO)** | read-write by ONE node (typical cloud block disk) — what I used |
| **ReadOnlyMany (ROX)** | read-only by many nodes |
| **ReadWriteMany (RWX)** | read-write by many nodes — needs NFS/CephFS/cloud file share |

## Deep question I raised: many pods writing to one volume — how are writes correct?
1. **K8s blocks it by default:** RWO = one node only. Multi-node writes need RWX + a networked
   filesystem.
2. **Even with RWX, a filesystem gives no transactions/isolation** — concurrent writers to the
   same files corrupt data, and each node caches independently (stale reads). Anti-pattern.
3. **Real architecture:** many **stateless** app Pods talk over the network to **one database**
   that owns the storage and handles concurrency (transactions, locks, isolation, WAL, cache
   coherence). Delegate write-correctness to software built for it — don't use a shared
   filesystem as a database.
4. **StatefulSet:** when pods must be stateful (the DB itself, Kafka, Cassandra), each pod gets
   its **own** private PVC (never shared) and the app handles replication.

> Rule: stateless pods share nothing and scale horizontally; shared mutable state lives in ONE
> system (a database) that owns concurrency.

## Concept I can explain in an interview
**Q: Why did data on a PVC survive a Pod restart but data in the container FS didn't?**
A: The container filesystem is ephemeral and dies with the Pod; a PVC binds to a PersistentVolume
whose lifecycle is independent of any Pod, so a new Pod mounting the same PVC sees the same data.

**Q: PV vs PVC vs StorageClass?** PV = the actual storage; PVC = a Pod's request/claim for storage;
StorageClass = dynamically provisions a PV to fulfill a PVC. Pods reference PVCs, decoupling them
from the underlying disk.
