# VProfile — GitOps CI/CD on Amazon EKS

End-to-end DevOps project demonstrating an application delivery workflow on AWS using **Terraform**, **Amazon EKS**, **Docker**, **Amazon ECR**, **Helm**, **Argo CD**, and GitHub-based automation.

> **Implementation status:** This repository documents the target architecture shown in the project diagram. Individual pipelines and cloud resources should be verified in their respective repositories before treating them as operational. At the time of documentation, the public `vprofile-infra` GitHub repository appears empty.

## Architecture

![VProfile GitOps Architecture](assets/architecture.png)

The diagram illustrates three connected workflows:

1. **Infrastructure:** Terraform validates and plans infrastructure changes, applies approved changes to AWS, detects drift, can send notifications to Slack, and has a separate destroy workflow.
2. **Application CI:** Changes to VProfile source code go through pull requests, Maven build and unit tests, Checkstyle, SonarQube code analysis / quality gates, Docker image build, and publication to Amazon ECR.
3. **GitOps delivery:** The newly built image tag is updated in the Helm configuration. Argo CD watches the Git repository, renders the chart, and synchronizes Kubernetes resources to Amazon EKS.

These flows represent the **intended architecture**; do not assume all steps are automated until their workflow definitions and successful runs are verified.

## Repositories

| Repository | Responsibility |
| --- | --- |
| [vprofile-app](https://github.com/meissa20/vprofile-app) | Application source, Docker files, and CI / code quality configuration |
| [vprofile-infra](https://github.com/meissa20/vprofile-infra) | Terraform-based AWS / EKS infrastructure (architecture target; public repo currently appears empty) |
| [vprofile-helm](https://github.com/meissa20/vprofile-helm) | Helm chart, Kubernetes deployment definitions, and GitOps deployment configuration |
| **This repository** | Project overview, architecture, setup and operational documentation |

## Technology stack

- **AWS:** Amazon EKS, Amazon ECR, and EC2 (shown for the quality-analysis component in the diagram)
- **Infrastructure as Code:** Terraform
- **Containers and orchestration:** Docker, Kubernetes
- **Kubernetes packaging and GitOps:** Helm, Argo CD
- **Build and quality:** Maven, Checkstyle, SonarQube
- **Version control / automation:** GitHub and Git-based workflows
- **Notifications:** Slack (illustrated for infrastructure notifications)

## Delivery flow

```text
Developer commits code / opens PR
                |
                v
          vprofile-app
                |
      Build and unit tests
                |
    Checkstyle + SonarQube
                |
        Build Docker image
                |
        Push to Amazon ECR
                |
       Update Helm image tag
                |
          vprofile-helm
                |
         Argo CD detects Git
                |
    Render Helm chart + sync
                |
            Amazon EKS
                |
     Deployment / Service / Ingress
```

Separately, infrastructure changes are managed from `vprofile-infra` through Terraform validate, plan, and apply workflows, with drift detection and notification steps represented in the architecture diagram.

## Kubernetes resources

The attached architecture includes an **Ingress**, **Service**, **Deployment**, **PersistentVolumeClaim**, and **Secret** for VProfile running in EKS. The actual resources and values are controlled by the chart manifests and installed configuration in `vprofile-helm`.

## Prerequisites

To reproduce the project, you generally need:

- An AWS account with sufficient permissions, and AWS CLI configured
- Terraform and access to the infrastructure source
- `kubectl` configured for the target EKS cluster
- Helm for chart linting and local rendering
- Argo CD installed in the cluster, with permission to deploy to the target namespace
- An ECR repository and a valid container image/tag referenced by the chart

## Deploying with Argo CD

> The following is a **starting example**, not a verified deployment configuration. Confirm the `chart/` directory, image values, dependencies, required Secrets, namespace, and cluster access before syncing.

Save an Argo CD Application manifest in the GitOps repository (for example, `vprofile-helm/argocd/application.yaml`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: vprofile
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/meissa20/vprofile-helm.git
    targetRevision: HEAD
    path: chart
    helm:
      releaseName: vprofile
  destination:
    server: https://kubernetes.default.svc
    namespace: vprofile
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

Apply once from a machine configured to access EKS:

```bash
kubectl apply -f argocd/application.yaml
kubectl get applications -n argocd
```

The example intentionally uses **manual sync**. In the Argo CD UI, review the proposed resources and click **Sync** when ready. After validating deployment behavior, you can enable automatic synchronization as a separate decision.

Check rollout and connectivity:

```bash
kubectl get pods,deploy,svc,ingress,pvc -n vprofile
kubectl describe application vprofile -n argocd
```

### Access Argo CD privately

From a workstation with EKS API connectivity:

```bash
kubectl port-forward -n argocd svc/argocd-server 9000:443
```

Open **https://localhost:9000** on the **same workstation**. The terminal stays occupied while the tunnel is active. To retrieve the initial admin password on a fresh standard installation (if the Secret still exists):

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
printf '\n'
```

Change the initial admin password after logging in. Avoid exposing the UI to the public internet solely for testing.

## Validation and troubleshooting

```bash
# Confirm cluster connection
kubectl cluster-info

# Verify Argo CD components
kubectl get pods -n argocd

# Inspect Application status
kubectl get applications -n argocd
kubectl describe application vprofile -n argocd

# Inspect app resources
kubectl get all -n vprofile

# Validate chart locally (from vprofile-helm repository root)
helm lint ./chart
helm template vprofile ./chart
```

If a synchronization fails, check the Argo CD Application conditions and events first; then inspect Pod events, image pull permissions, Helm values, Secret references, and persistent storage provisioning as applicable.

## Security and operations notes

- Store credentials in a secure secrets solution; **never commit plaintext secrets, kubeconfig tokens, AWS keys, or Terraform state** to Git.
- Use least-privilege IAM and Kubernetes RBAC. ECR pulls, Terraform applies, CI pushes, and Argo CD synchronization require different permissions.
- Use remote protected Terraform state and reviewed plans for infrastructure changes.
- Control production changes through pull requests and protected branches; consider image digests or immutable tags.
- Treat Terraform destroy and Argo CD pruning as destructive operations requiring explicit safeguards.
- AWS resources (including EKS, EC2, ECR, and load balancers) may incur ongoing charges.

## Project status / next steps

- [x] Overall architecture diagram provided
- [x] Application and Helm repositories publicly visible
- [ ] Verify / publish Terraform code and workflows in `vprofile-infra`
- [ ] Verify the exact CI pipeline definitions and successful quality-gate runs
- [ ] Verify automated image-tag updates in `vprofile-helm`
- [ ] Confirm Argo CD sync and a successful EKS rollout
- [ ] Add screenshots of GitHub Actions, SonarQube, Argo CD, and the deployed app
- [ ] Document production-ready secrets, state, network, and access design

## Architecture source

Diagram reproduced from the architecture PDF supplied with the project. It is the primary reference for the workflow described above.
