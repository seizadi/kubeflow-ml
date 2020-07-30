# KubeFlow-ML

## Setup repo
This repo assuems that a cluster was setup using eksctl,
[git@github.com:seizadi/kubeflow.git repo](https://github.com/seizadi/kubeflow). 
The use of this Git repo was started with call in that repo to:
```bash
make repo
```
We then programmed Flux to sync this repo with the cluster.

Any changes to the repo will be picked up and applied to cluster, so
verify kustomize is working before commit
```bash
kubectl apply --dry-run -k .
```
You can see the deployed manifests:
```bash
kustomize build . | more
```

When we commit the changes, we can use fluxctl sync to force change
rather than wait for the poll cycle to apply:
```bash
$ git push
$ fluxctl sync --k8s-fwd-ns flux
```
