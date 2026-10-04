# GitOps

Each service has its own Argo CD Application in `argocd/application.yaml`.
Applications deploy directly from `base/<service>` into `idt-demo`; there are no overlays.
The `idt-rds-config` application deploys the RDS CA ConfigMap from `config/rds`.

## RDS connection

This config connects auth to an existing PostgreSQL RDS instance. It does not create an AWS database.

1. Copy `config/rds/secret.yaml.example` to `config/rds/secret.yaml` (ignored by Git).
2. Replace the database endpoint, database name, username, password, and JWT secret. URL-encode special characters in the username and password.
3. Create the namespace and apply the secret locally:

   ```sh
   kubectl create namespace idt-demo --dry-run=client -o yaml | kubectl apply -f -
   kubectl apply -f config/rds/secret.yaml
   ```

The `idt-auth` Secret is shared by auth and backend and is deliberately not managed in Git.
Auth mounts the `rds-ca` ConfigMap at `/certs/rds-ca.pem` and verifies the database certificate.
Allow cluster traffic to the RDS endpoint on port 5432 in the RDS security group.

The CA bundle comes from AWS: https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem.
To refresh it:

```sh
curl --fail --location https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem -o config/rds/global-bundle.pem
```

## Argo CD

The manifests assume this directory is `deployment/gitops` in
`https://github.com/Devops-communityy/industrial-digital-twin.git` on branch `main`.
Change `repoURL`, `targetRevision`, and `path` in the applications, and `sourceRepos` in the project if your repository differs.

Push the manifests, then register the applications with an existing Argo CD installation:

```sh
kubectl apply -k argocd
```

Sync `idt-rds-config` before auth starts. Independent applications do not enforce sync ordering.
Replace the placeholder simulator image in `base/simulator/deployment.yaml` before syncing `idt-simulator`.

If `industrial-digital-twin-demo` is already registered, delete only its Application without cascading resource deletion before syncing the new applications. Do not let the old application prune workloads during the handover.