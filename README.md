# MLOps ArgoCD Demo (Terraform + ArgoCD + Helm + MLflow)

This repo contains Terraform to install **Argo CD** into an existing **AWS EKS** cluster and a companion Git repo with a minimal **MLflow** Helm chart managed by Argo CD (GitOps).

- **EKS cluster:** `ml-eks` (region `eu-central-1`)
- **Argo CD namespace:** `infra-tools`
- **App repo (Helm chart + Application):** https://github.com/tetyanaCY/mlflow-helm-app
- **Service exposure:** via `kubectl port-forward`


---

## 1) Prerequisites

- AWS CLI, kubectl, Helm, Terraform ≥ 1.5
- An existing EKS cluster and `kubectl` context pointing to it
- (Optional) S3 bucket if you plan to use remote Terraform state

Quick EKS context check:
```powershell
aws sts get-caller-identity
aws eks update-kubeconfig --region eu-central-1 --name ml-eks
kubectl get nodes
```

Repo structure you should have locally:
```
mlops-argocd-demo/
├── argocd/
│   ├── main.tf
│   ├── outputs.tf
│   ├── terraform.tf
│   ├── variables.tf
│   └── values/
│       └── argocd-values.yaml
```

---

## 2) Deploy Argo CD with Terraform

From the `argocd/` folder:
```powershell
cd argocd
terraform init
terraform apply -auto-approve `
  -var="aws_region=eu-central-1" `
  -var="cluster_name=ml-eks"
```

Verify:
```powershell
kubectl get pods -n infra-tools
kubectl get svc  -n infra-tools
```

Expected pods: `argocd-server`, `argocd-repo-server`, `argocd-application-controller`, etc.  
Service `argocd-server` should be `ClusterIP` (port 80).

---

## 3) Open Argo CD UI

Port-forward (keep this terminal open):
```powershell
kubectl -n infra-tools port-forward svc/argocd-server 8080:80
# open http://localhost:8080
```

Get the initial admin password (PowerShell friendly):
```powershell
$raw = kubectl -n infra-tools get secret argocd-initial-admin-secret -o jsonpath='{.data.password}'
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($raw))
```
Login with **username:** `admin` and the decoded **password**.

---

## 4) App Git repo and Argo CD Application

The MLflow Helm chart and Application manifest live in a separate repo:
- **Repo:** https://github.com/tetyanaCY/mlflow-helm-app
- **Helm path:** `helm/mlflow`
- **Application:** `application.yaml` (either in that repo or applied locally)

Apply the Application to Argo CD:
```powershell
# from the mlflow-helm-app repo folder (or use an absolute path)
kubectl apply -n infra-tools -f application.yaml
```

The Application has:
- `syncPolicy.automated` with `prune` and `selfHeal`
- `syncOptions: CreateNamespace=true`
- Destination namespace: `mlflow`

---

## 5) Verify the deployment

```powershell
kubectl get ns
kubectl get pods -n mlflow
kubectl get svc  -n mlflow
```

You should see a `Deployment` pod and a `Service` named `mlflow` on port **5000**.

---

## 6) Open MLflow

```powershell
kubectl -n mlflow port-forward svc/mlflow 5000:5000
# open http://localhost:5000
```

If you see connection refused, ensure the pod is `Running/Ready` and check logs:
```powershell
$POD = kubectl get pods -n mlflow -l app.kubernetes.io/name=mlflow -o jsonpath='{.items[0].metadata.name}'; echo $POD
kubectl logs $POD -n mlflow --tail=200
```

---

## 7) Troubleshooting (handy tweaks)

- **Argo CD UI shows “Failed to load data”**  
  Keep the port-forward running, Hard refresh the page (Ctrl+F5), or clear site data and re-login.

- **MLflow pod OOMKilled / CrashLoopBackOff**  
  Use the lightweight UI process and limit Gunicorn workers:
  - In the chart’s Deployment args, use `mlflow ui` (not `server`).
  - Set `env`:
    ```yaml
    - name: GUNICORN_CMD_ARGS
      value: "--workers 1 --timeout 120"
    ```
  - In `values.yaml`, raise memory:
    ```yaml
    resources:
      limits:
        memory: "1Gi"
      requests:
        memory: "512Mi"
    ```
  - Optionally pin a stable tag: `image.tag: v2.14.1`.

- **Private Git repo**  
  In Argo CD UI → **Settings → Repositories → Connect** → add the Git repo with HTTPS + PAT or SSH.

---

## 8) Clean up (avoid costs!)

To remove the app (optional):
```powershell
kubectl delete -n infra-tools -f application.yaml
```

To remove Argo CD (from `argocd/`):
```powershell
terraform destroy -auto-approve `
  -var="aws_region=eu-central-1" `
  -var="cluster_name=ml-eks"
```


---

## 9) Links

- **App repo (Helm + Application):** https://github.com/tetyanaCY/mlflow-helm-app
- **Argo CD:** https://argo-cd.readthedocs.io/
- **MLflow:** https://mlflow.org/
