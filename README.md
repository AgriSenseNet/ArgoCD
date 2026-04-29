# gitops-apps

ArgoCD GitOps repository for the Kafka-on-Kubernetes stack.

ArgoCD installed by `infra-ansible` points to this repo. Changes merged to `main` are automatically synced to the cluster.

## Repository Structure

```
applications/          # ArgoCD Application manifests (app-of-apps pattern)
├── strimzi-operator/  # Strimzi Kafka Operator (sync-wave 0)
├── kafka-cluster/     # Kafka cluster CR (sync-wave 1)
├── storage/           # StorageClasses (sync-wave -1, must run first)
├── monitoring/        # Prometheus + Grafana (sync-wave 2)
└── ingress-nginx/     # Nginx Ingress Controller (sync-wave 1)

manifests/             # Actual Kubernetes manifests referenced by Applications
├── kafka/
│   ├── kafka-cluster.yaml    # Kafka CR (KRaft mode, 3 brokers)
│   └── example-topic.yaml    # Example KafkaTopic CR
├── storage/
│   └── storageclasses.yaml   # gp3 StorageClasses (kafka-gp3, standard-gp3)
└── monitoring/
    ├── kube-prometheus-stack.yaml  # ArgoCD Application for Prometheus/Grafana
    └── strimzi-pod-monitor.yaml    # PodMonitor for Strimzi Kafka metrics
```

## Deployment Order (sync-wave)

| Wave | Resource |
|---|---|
| -1 | StorageClasses (must exist before PVCs) |
| 0 | Strimzi Kafka Operator |
| 1 | Kafka Cluster CR + Ingress Nginx |
| 2 | Monitoring stack |

## Key Design Decisions

- **KRaft mode** — No ZooKeeper. Requires Strimzi 0.40+ and Kafka 3.6+.
- **prune: false on kafka-cluster** — Prevents ArgoCD from deleting the Kafka CR on Git changes, protecting data.
- **StorageClass `kafka-gp3`** has `reclaimPolicy: Retain` — PVs survive PVC deletion.
- **3 replicas, min.insync.replicas=2** — Strong durability defaults.
- **NodePort external listener** — For client access from within the VPC without a load balancer.

## Updating the Cluster

1. Edit manifests in this repo.
2. Open a Pull Request to `main`.
3. After merge, ArgoCD auto-syncs within ~3 minutes (or trigger manually via UI/CLI).

```bash
# Force immediate sync
argocd app sync kafka-cluster
```

## Accessing ArgoCD

```bash
# Port-forward to ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Login (get initial password from Ansible output or:)
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

## Adding a New Application

1. Create a Helm chart or manifest directory under `manifests/`.
2. Add a new `Application` manifest under `applications/`.
3. Set the appropriate `sync-wave` annotation.
4. Commit and push — ArgoCD picks it up automatically.
