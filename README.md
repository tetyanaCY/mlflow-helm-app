# MLOps ArgoCD Demo (Terraform + ArgoCD + Helm + MLflow)

## Stack
- AWS EKS: `ml-eks` (region `eu-central-1`)
- ArgoCD in namespace `infra-tools` via Terraform Helm release
- App repo (Helm + Application): https://github.com/tetyanaCY/mlflow-helm-app

---

## 1) Deploy ArgoCD with Terraform
```bash
cd argocd
terraform init
terraform apply -auto-approve \
  -var="aws_region=eu-central-1" \
  -var="cluster_name=ml-eks"
