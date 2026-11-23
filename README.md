# DevOps Lab 4: Advanced Deployment Strategies and Enterprise DevOps

Kubernetes (Minikube) experiments on advanced deployment strategies and secure delivery. Each experiment has its own folder with its manifests and commands.

**Author:** Akshita Sharma ([@Akshita04987](https://github.com/Akshita04987))
**Course:** DevOps and Automation Lab (ENSP461), Lab 4

## Experiments
| Item | Experiment | Folder | Status |
|---|---|---|---|
| 1 | Disaster recovery (backup, restore, failover) | `item1-disaster-recovery/` | Completed |
| 2 | Automated testing in the CI/CD pipeline | `item2-ci-testing/` | Planned |
| 3 | Blue-Green deployment | `item3-blue-green/` | Completed |
| 4 | GitOps-based delivery | `item4-gitops/` | Planned |
| 6 | Secret management | `item6-secret-management/` | Completed |
| 7 | Canary deployment | `item7-canary/` | Completed |

(Item 5 is the written report, which is submitted on the course portal.)

## Prerequisites
Docker Desktop, Minikube and kubectl. Start the cluster with `minikube start`.

## Item 1: Disaster recovery
A stateful app (Deployment with a PersistentVolumeClaim) stores orders in `/data/records.txt`. The experiment demonstrates backup, restore after a total loss, and failover.

```bash
kubectl apply -f item1-disaster-recovery/dr-app.yaml

# Backup: manifests and data
kubectl get deployment/dr-app pvc/dr-data -o yaml > backup/manifests-backup.yaml
kubectl cp "<pod>:/data/records.txt" backup/records.txt

# Disaster: delete the app and its volume
kubectl delete -f item1-disaster-recovery/dr-app.yaml

# Restore: re-create the app, then put the backed-up data back
kubectl apply -f item1-disaster-recovery/dr-app.yaml
kubectl cp backup/records.txt "<pod>:/data/records.txt"

# Failover: delete the pod, the Deployment starts a new one with the same data
kubectl delete pod <pod>
```

| Objective | Target | Result |
|---|---|---|
| RPO (data loss) | Time since last backup | 0 orders lost (backup taken after the last write) |
| RTO (time to recover) | Under 2 minutes | Restore 46 seconds, failover under 30 seconds |

Recommendations for production: automated and off-site backups (CronJob or Velero), encrypted backups, regular restore tests and multi-node high availability.

## Item 3: Blue-Green deployment
Two identical deployments run side by side: **blue** (version 1.0) and **green** (version 2.0). One Service decides which version receives the traffic through its selector.

```bash
kubectl apply -f item3-blue-green/blue-deployment.yaml
kubectl apply -f item3-blue-green/green-deployment.yaml
kubectl apply -f item3-blue-green/service.yaml

# Which version do users see?
kubectl run curl-test --rm -i --restart=Never --image=curlimages/curl -- curl -s http://bluegreen-service

# Switch all traffic to green (no downtime)
kubectl patch service bluegreen-service -p '{"spec":{"selector":{"app":"bluegreen","version":"green"}}}'
```

Result: before the switch the service answered `BLUE version 1.0`, after the switch it answered `GREEN version 2.0`.

| Advantages | Limitations |
|---|---|
| Zero downtime | Needs double the resources |
| Instant switch and instant rollback | All users move at once |
| New version can be tested before users see it | Database changes are difficult |

## Item 6: Secret management
Credentials are stored in a Kubernetes Secret and are never written in YAML files or Git. The pod reads them as environment variables.

```bash
kubectl create secret generic db-credentials --from-literal=username=<user> --from-literal=password=<password>
kubectl apply -f item6-secret-management/secret-demo.yaml
kubectl logs secret-demo
```

Result: the pod printed `User=labuser` and `Password-length=13`, so it can use the credentials without exposing the password.

Safe handling used here:
- The real values are created in the cluster and are **not stored in the repository**.
- `.gitignore` excludes `secrets/`, `*.env` and `*.secret.yaml`.
- `kubectl describe secret` shows only the size of each value.
- Kubernetes Secrets are only base64-encoded. For production, use encryption at rest and a secret store such as HashiCorp Vault or a cloud secret manager.

## Item 7: Canary deployment
A few pods run the new version while most pods run the stable version. The Service selects both through the shared label `app: canary-demo`, so the traffic share follows the pod count.

```bash
kubectl apply -f item7-canary/stable-deployment.yaml     # 4 stable pods
kubectl apply -f item7-canary/canary-deployment.yaml     # 1 canary pod (about 20% of traffic)
kubectl apply -f item7-canary/canary-service.yaml

# Gradual rollout
kubectl scale deployment app-canary --replicas=2
kubectl scale deployment app-stable --replicas=3

# Rollback: remove the canary
kubectl scale deployment app-canary --replicas=0
kubectl scale deployment app-stable --replicas=4
```

Result: in a test of 20 requests both versions answered (stable `STABLE version 1.0`, canary `CANARY version 2.0`). After the rollback every answer was `STABLE version 1.0`.

Typical rollback triggers: higher error rate, higher latency, failing readiness checks, restarting pods or negative user feedback.
