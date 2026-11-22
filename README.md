# DevOps with Kubernetes 2026: project configuration

Kubernetes configuration of the course project (the todo app). The application code is in https://github.com/felixwickholm/devops-with-kubernetes. This split is exercise 4.10: code and configuration live in different repositories.

ArgoCD deploys this repository. Nobody applies the project by hand.

## Layout

- `apps/`: manifests of each service, without a namespace
  - `todo-app`: the web page and its image cache
  - `todo-backend`: the todo API and its Postgres StatefulSet
  - `broadcaster`: the NATS subscriber that forwards todo messages
  - `todo-cronjob`: the hourly Wikipedia todo
- `base/`: all apps together
- `overlays/staging/`, `overlays/production/`: the environments (namespace, differences, image tags)
- `argocd/`: the ArgoCD Applications, one per environment
- `secrets/`: the database Secret, encrypted with SOPS (age). The age key is not in any repository.

| | staging | production |
|---|---|---|
| Image tags set by | every commit to `main` of the code repository | every tag of the code repository |
| ArgoCD Application | `argocd/project-staging.yaml` | `argocd/project-production.yaml` |
| Broadcaster | only logs the messages | forwards the messages (`webhook-receiver.yaml`) |
| Database backup | none | daily `pg_dump` CronJob (`backup-cronjob.yaml`) |
| Address | http://staging.localhost:8081 | http://localhost:8081 |

## How an update reaches the cluster

1. Code changes are pushed to the code repository.
2. Its GitHub Actions workflows build the images (amd64 and arm64, tagged with the commit SHA) and push them to Docker Hub.
3. The workflow checks out this repository, sets the new tags with `kustomize edit set image` in `overlays/staging` (commit to main) or `overlays/production` (tag), and commits here.
4. ArgoCD follows `main` of this repository and deploys the change.

Configuration changes (replicas, ConfigMaps and so on) are committed here directly, and ArgoCD deploys them.

The workflows in the code repository need the secrets `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN` and `CONFIG_REPO_TOKEN` (a fine-grained token with write access to the contents of this repository). Without them they skip the build with a notice.

## Run

Once per cluster: NATS and ArgoCD (see the code repository), and the database Secret:

```sh
export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt
for env in staging production; do
  kubectl create namespace $env
  sops --decrypt secrets/todo-postgres-secret.enc.yaml | kubectl apply -n $env -f -
  kubectl apply -f argocd/project-$env.yaml
done
```
