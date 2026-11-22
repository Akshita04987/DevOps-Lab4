# DevOps Lab 4: Advanced Deployment Strategies and Enterprise DevOps

Kubernetes (Minikube) experiments for advanced deployment strategies and secure delivery: Blue-Green deployment, Canary deployment and secret management.

**Author:** Akshita Sharma ([@Akshita04987](https://github.com/Akshita04987))
**Course:** DevOps and Automation Lab (ENSP461), Lab 4

---

## Project structure
```text
DevOps-Lab4/
├── blue-deployment.yaml      # Blue version 1.0 (2 replicas)
├── green-deployment.yaml     # Green version 2.0 (2 replicas)
├── service.yaml              # Service that switches between blue and green
├── stable-deployment.yaml    # Stable version 1.0 (4 replicas)
├── canary-deployment.yaml    # Canary version 2.0 (1 replica)
├── canary-service.yaml       # Service that sends traffic to stable and canary
├── secret-demo.yaml          # Pod that reads credentials from a Kubernetes Secret
├── .gitignore                # Keeps secrets out of Git
└── README.md
```

## Prerequisites
Docker Desktop, Minikube and kubectl. Start the cluster with `minikube start`.

## 1. Blue-Green deployment
Two identical deployments run side by side: **blue** (version 1.0) and **green** (version 2.0). One Service decides which version receives the traffic through its selector.

```bash
kubectl apply -f blue-deployment.yaml
kubectl apply -f green-deployment.yaml
kubectl apply -f service.yaml

# Which version do users see?
kubectl run curl-test --rm -i --restart=Never --image=curlimages/curl -- curl -s http://bluegreen-service

# Switch all traffic to green (no downtime)
kubectl patch service bluegreen-service -p '{"spec":{"selector":{"app":"bluegreen","version":"green"}}}'

# Roll back to blue
kubectl patch service bluegreen-service -p '{"spec":{"selector":{"app":"bluegreen","version":"blue"}}}'
```

| Advantages | Limitations |
|---|---|
| Zero downtime | Needs double the resources |
| Instant switch and instant rollback | All users move at once |
| New version can be tested before users see it | Database changes are difficult |

## 2. Canary deployment
A small number of pods run the new version while most pods run the stable version. The Service selects both through the shared label `app: canary-demo`, so the traffic share follows the pod count.

```bash
kubectl apply -f stable-deployment.yaml      # 4 stable pods
kubectl apply -f canary-deployment.yaml      # 1 canary pod (about 20% of traffic)
kubectl apply -f canary-service.yaml

# Gradual rollout
kubectl scale deployment app-canary --replicas=2
kubectl scale deployment app-stable --replicas=3

# Rollback: remove the canary
kubectl scale deployment app-canary --replicas=0
kubectl scale deployment app-stable --replicas=4
```

Typical rollback triggers: higher error rate, higher latency, failing readiness checks, restarting pods, or negative user feedback.

## 3. Secret management
Credentials are stored in a Kubernetes Secret and never written in YAML files or Git. The pod reads them as environment variables.

```bash
kubectl create secret generic db-credentials --from-literal=username=<user> --from-literal=password=<password>
kubectl apply -f secret-demo.yaml
kubectl logs secret-demo
```

Safe handling practices used here:
- The secret is created directly in the cluster and the real values are **not stored in the repository**.
- `.gitignore` excludes `secrets/`, `*.env` and `*.secret.yaml`.
- `kubectl describe secret` shows only the size of each value, not the value.
- The pod prints only the user name and the **length** of the password, never the password.
- Kubernetes Secrets are only base64-encoded. For production use, enable encryption at rest and a secret store such as HashiCorp Vault or a cloud secret manager.
