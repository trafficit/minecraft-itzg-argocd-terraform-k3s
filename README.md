# minecraft-itzg-argocd-terraform-k3s
minecraft99nights-itzg-argocd-terraform-k3s


Self-hosted Minecraft server managed with GitOps principles using k3s, ArgoCD, Terraform and the itzg/minecraft-server Helm chart.

## Purpose

This repository contains everything needed to run a persistent Minecraft server (Paper / Vanilla / etc.) in a single-node k3s cluster with:

- declarative infrastructure (Terraform)
- GitOps deployment (ArgoCD)
- automatic world backups to S3 (other varios)
- restore from latest backup on pod start
- private cluster access via WireGuard
- basic ingress exposure

It is intended both as a personal server setup and as a reference implementation for GitOps + stateful workloads on lightweight Kubernetes.

## Repository structure

    .
    ├── terraform/                  AWS resources: S3 bucket + IAM user for backups
    │   └── main.tf
    ├── argocd-apps/                ArgoCD Application manifests
    │   ├── minecraft-app.yaml
    │   └── cronjob-app.yaml
    ├── minecraft-values/           Custom values for itzg/minecraft-server chart
    │   └── values.yaml
    ├── cronjob/                    Backup CronJob manifest
    │   └── backup-cronjob.yaml
    ├── manifests/                  Optional raw manifests (wireguard, extras)
    └── HELP.md                     this file

## Prerequisites

- Linux machine / VPS with public IP (for k3s + ingress)
- AWS account (S3 + IAM)
- Domain name + ability to create A record (for ingress)
- Installed locally:
  - terraform >= 1.5
  - kubectl
  - helm >= 3.10
  - aws cli v2
  - git
  - (optional) argocd cli, gh cli, wireguard-tools

## Quick start order

1. Clone this repository
   git clone https://github.com/<your-username>/minecraft-itzg-argocd-terraform-k3s.git
   cd minecraft-itzg-argocd-terraform-k3s

2. Create and apply Terraform infrastructure
   cd terraform
   terraform init
   terraform apply

   → save bucket name + access key + secret key from output

3. Install k3s 
   curl -sfL https://get.k3s.io | sh -
   # copy kubeconfig to local ~/.kube/config and fix server address

4. Install ArgoCD
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

5. Create AWS credentials secret in minecraft namespace
   kubectl create namespace minecraft
   kubectl create secret generic aws-credentials \
     -n minecraft \
     --from-literal=AWS_ACCESS_KEY_ID=... \
     --from-literal=AWS_SECRET_ACCESS_KEY=...

6. Update values.yaml
   - set correct S3 bucket name in initContainers env
   - change rcon password, motd, server type/version, ingress host etc.

7. Commit & push changes
   git add .
   git commit -m "initial configuration"
   git push

8. Create ArgoCD Applications
   kubectl apply -f argocd-apps/

9. Sync applications in ArgoCD UI or via CLI
   argocd app sync minecraft
   argocd app sync minecraft-backup-cron

10. (Optional) Set up WireGuard tunnel for private kubectl & game access

## Accessing the server

- Public: mc.wizard.dedyn.io:25555     nginx -> (via ingress)
- Private: connect to server IP through WireGuard tunnel wg2

## Backup & restore logic

- Init container (amazon/aws-cli) downloads the latest .tar.gz from S3 → extracts to /data
- CronJob (every 4 hours) creates timestamped archive → uploads to S3 + overwrites latest.tar.gz

## Important notes

- Single-node k3s → no HA by design
- local-path storage class used by default → consider longhorn / openebs for better durability
- Secrets (rcon password, AWS keys) are currently plain → replace with Sealed Secrets / External Secrets in production
- No monitoring / alerting included yet

## Next steps (possible improvements)

- Add cert-manager + Let's Encrypt
- Install longhorn / openebs for replicated storage
- Add Prometheus + Minecraft exporter + Grafana
- Velocity / mc-router for multiple servers / worlds
- Point-in-time restore logic
- Retention policy for old backups

Feedback and PRs welcome.
