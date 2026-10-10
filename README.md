# GitOps

Each workload has its own Argo CD Application under its service directory (`backend/app.yaml`, `auth/app.yaml`, `frontend/app.yaml`, `kafka/app.yaml`, `mqtt/app.yaml`, and `simulator/app.yaml`). Each points to its matching `base/<service>` Kustomization. The root `kustomization.yaml` registers these Applications and the shared configuration Application. There are no overlays.

RDS is an external AWS database, not a Kubernetes Deployment in this repository. The `auth` workload connects to it; there is no separate `db` workload to deploy.

## Configure

Edit `config/kustomization.yaml` for shared non-secret settings such as `CORS_ORIGINS`. The `idt-config` Application is the sole owner of the shared ConfigMap, including the RDS CA bundle and Mosquitto configuration.

Create the one shared `idt-auth` Secret locally; it is intentionally not stored in Git:

1. Copy `config/rds/secret.yaml.example` to `config/rds/secret.yaml` (ignored by Git).
2. Set `DATABASE_URL` for RDS and a long random `JWT_SECRET`. URL-encode special characters in the database username and password.
3. Create the namespace and apply the Secret:

   ```sh
   kubectl create namespace idt-demo --dry-run=client -o yaml | kubectl apply -f -
   kubectl apply -f config/rds/secret.yaml
   ```

Auth and backend share `JWT_SECRET`; auth also reads `DATABASE_URL` and verifies the RDS certificate from `idt-config` at `/certs/rds-ca.pem`. Allow cluster traffic to the RDS endpoint on port 5432 in the RDS security group.

Replace the placeholder simulator image in `base/simulator/deployment.yaml` before syncing.

The RDS CA bundle comes from https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem. Refresh it with:

```sh
curl --fail --location https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem -o config/rds/global-bundle.pem
```

## Deploy

The manifests assume this repository is `deployment/gitops` in `https://github.com/Devops-communityy/industrial-digital-twin.git` on branch `main`. Update each `app.yaml` and `argocd/project.yaml` if the repository or branch changes.

After pushing the manifests and creating the Secret, register the project and all Applications:

```sh
kubectl apply -k .
```

Argo CD shows one Application per service, plus `idt-config`. They sync independently; the shared ConfigMap appears when `idt-config` syncs. If replacing the previous aggregate Application, remove only that Application with orphan cascading before applying this layout so its workloads are preserved:

```sh
kubectl delete application -n argocd --cascade=orphan --ignore-not-found industrial-digital-twin
kubectl delete application -n argocd --cascade=orphan --ignore-not-found idt-rds-config
```