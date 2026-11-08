
## Problem 1 – Service interruptions

### A – Releases and node maintenance

**Risk**

- **one replica** - When there is only one replica, downtime occurs during its removal (e.g. node drain during maintenance), the API will go down until a new pod comes up.


- **no readiness or startup probe** - Without a readiness or startup probe, Kubernetes has no way of knowing when a new pod is truly ready. As a result, it immediately puts the new pod into the Service - causing requests to it to fail, and deletes the previous pod too soon, assuming the new one is already ready.

- **no resource requests/limits** - Without requests, K8S doesn't know how much CPU and memory to reserve, and might place the pod on an overloaded node.
Without limits, a memory leak in the API or another pod could consume the node’s memory and crash the API.


- **no graceful shutdown** - when the old pod is deleted, it may shut down before the load balancer becomes aware of it. As a result, the load balancer will continue to send requests to the pod - requests that will subsequently fail. Also, any requests already in progress will be interrupted too.

- **no PodDisruptionBudget** - There is no safeguard to ensure that not all replicas are taken down together, which could leave the system without a response during node maintenance.

**Fix**

Deployment excerpt (only the changed fields) and a PodDisruptionBudget:

```yaml
spec:
  replicas: 2
  strategy:
    rollingUpdate:
      maxSurge: 1                        # start the new pod first
      maxUnavailable: 0                  # never drop below 2 ready pods
  template:
    spec:
      terminationGracePeriodSeconds: 60  # 30s preStop + in-flight requests + margin
      containers:
        - name: api
          resources:                     # example values, tune from monitoring
            requests: { cpu: 250m, memory: 256Mi }
            limits:   { memory: 512Mi }
          startupProbe:                  # allow slow startup
            httpGet: { path: /healthz, port: 8000 }
            failureThreshold: 24
            periodSeconds: 5
          readinessProbe:                # send traffic only when ready
            httpGet: { path: /healthz, port: 8000 }
            periodSeconds: 5
          lifecycle:
            preStop:                     # let the LB stop routing before SIGTERM (tune from measured NEG drain latency)
              exec: { command: ["sleep", "30"] }
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: { name: api }
spec:
  minAvailable: 1                        # node drain keeps one pod running
  selector: { matchLabels: { app: api } }
```
Assumes the API exposes a lightweight `/healthz` endpoint.

**Safe implementation**

1. Add the `/healthz` endpoint to the API and release it first, on its own.
2. Check that both nodes have room for 3 API pods during a rollout (requests × 3 must fit the nodes' allocatable resources: `kubectl describe nodes`).
3. Apply the Deployment change through the pipeline and watch `kubectl rollout status deployment/api`; roll back with `kubectl rollout undo deployment/api` if needed.
4. Apply the PDB only after 2 replicas are Ready (with 1 replica, `minAvailable: 1` would block node drains).

**Verification** (proposed, not tested on a real cluster)

- **Release:** run continuous load (hey) against the API, run `kubectl rollout restart deployment/api`, expect zero 5XX responses.
- **Node maintenance:** under the same load, run `kubectl drain <node> --ignore-daemonsets`, expect the PDB to keep one pod serving and zero 5XX responses.
- **Bad release:** deploy an image that fails the readiness probe, expect the rollout to stop, old pods keep serving, and `kubectl rollout status` to time out.

### B – Zone outage

**Risk**

- **GKE nodes in one zone** - both nodes are in the same zone, so a zone outage stops every API pod, even with the 2 replicas from part A. The HTTPS load balancer keeps running, but it has no healthy backends and returns 502.
- **Single-zone Cloud SQL** - the database is unavailable until the zone recovers.
- **Not affected** - the Cloud Storage bucket and Artifact Registry are regional (data is stored across zones). The pipeline agent VM is zonal; losing it blocks deployments, not users.

**Minimum changes**

- **GKE:** add a second zone to the node pool (`--node-locations me-west1-a,me-west1-b`, one node per zone), sized so one zone can carry the full load. Spread the API pods across zones:

```yaml
topologySpreadConstraints:                 # one API pod per zone
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway      # still schedule if a zone is down
    labelSelector: { matchLabels: { app: api } }
```

Apply the same replicas, probes and spread to the web frontend, so users can still reach the site.

- **Cloud SQL:** enable High Availability (regional instance): a standby in another zone with synchronous replication and automatic failover to the same private IP (about one minute of downtime). The API must retry database connections.
