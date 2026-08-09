# Runbook - Azure Kubernetes Service (AKS)

> Operational runbook for the Aetherion AirOps platform. Cluster: `aetherion-aks`,
> namespace: `aetherion`.

## Common symptoms & first checks

| Symptom | First check |
|---------|-------------|
| A service tile is red in Ops Center | `kubectl get pods -n aetherion` and pod logs |
| Pod `CrashLoopBackOff` | `kubectl describe pod` + `kubectl logs --previous` |
| Pod `Pending` | Node capacity / autoscaler: `kubectl get nodes`, events |
| High latency (amber tiles) | Application Insights latency + CPU/HPA state |
| Readiness failing but process up | Downstream dependency (PostgreSQL/Redis) |

## Triage commands

```
kubectl get pods -n aetherion -o wide
kubectl describe deploy/<service> -n aetherion
kubectl logs deploy/<service> -n aetherion --tail=200
kubectl get hpa -n aetherion
kubectl get events -n aetherion --sort-by=.lastTimestamp
```

## Pods crashing / restarting

- Symptom: repeated restarts, red tile, liveness failing.
- Investigate: `kubectl describe pod`, `kubectl logs --previous`, and the recent
  rollout history (`kubectl rollout history deploy/<svc> -n aetherion`).
- Remediate: if a recent change caused it, roll back to the last good revision
  with `kubectl rollout undo deploy/<svc> -n aetherion` (reversible). Otherwise
  fix the failing dependency or configuration and redeploy.

## Memory pressure / OOM

- Symptom: restarts with reason `OOMKilled`, rising memory in Azure Monitor.
- Remediate: raise memory limits or scale out replicas; investigate for leaks in
  recent changes.

## Elevated latency

- Symptom: amber tiles, elevated request duration in Application Insights.
- Investigate: downstream dependencies (PostgreSQL/Redis) and CPU/HPA state.
- Remediate: let the HPA scale; resolve the slow dependency.

## Elevated error rate

- Symptom: failed requests (HTTP 5xx) in Application Insights, red/amber tile.
- Investigate: correlate with recent deployments and change history.
- Remediate: `kubectl rollout undo deploy/<svc> -n aetherion` to the last good
  revision.

## Guardrails (AKS-specific)

- Do **not** cordon/drain nodes or restart node pools during business hours.
- Pod restarts and `rollout restart`/`undo` are safe at any time.
- Prefer scaling replicas and letting the HPA/cluster-autoscaler respond over
  manual node operations.
