# StreamingVideo-config

GitOps configuration for [StreamingVideo](https://github.com/22127378/StreamingVideo). ArgoCD watches this repository; nothing here is applied by hand except the root Application.

## Layout

```
apps/                  child Applications (App-of-Apps), synced by the root Application
  cluster.yaml         NodePools, PriorityClasses, IngressClass, StorageClass   wave -2
  keda.yaml            KEDA Helm chart                                          wave -1
  argo-rollouts.yaml   Argo Rollouts Helm chart                                 wave -1
  streamforge.yaml     the application, from envs/dev                           wave 0
envs/dev/
  kustomization.yaml   base manifests from the code repo pinned to a commit, image tags, AWS IDs
  config.env           ConfigMap streamforge-config (written by k8s/deploy.sh)
  migrate.env          ConfigMap for the database migration Jobs
  kustomizeconfig.yaml name references inside KEDA and Argo Rollouts resources
```

The root Application lives in the code repository (`k8s/argocd/root-app.yaml`) and is applied once by `k8s/argocd/install.sh`. It points at `apps/`.

## How a change reaches the cluster

1. A push to `main` in the code repository runs CI: tests, Checkov, image build and Trivy scan.
2. CI assumes an AWS role through GitHub OIDC and pushes the images to ECR, tagged with the commit SHA.
3. CI commits to this repository: every `newTag` and the `ref=` of both remote bases become that commit. Manifests and images always come from the same commit.
4. ArgoCD syncs `envs/dev`: migrations run as a Sync hook, then Argo Rollouts runs a canary for each HTTP service and rolls back on its own if the Prometheus analysis fails.

Rolling back by hand is a `git revert` of the CI commit here.

## Access

CI writes with a deploy key that has write access to this repository only (`CONFIG_REPO_DEPLOY_KEY` secret in the code repository). The repository is public, so ArgoCD needs no credentials. If you make it private, add a repository secret in the `argocd` namespace.
