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
kubectl apply --dry-run=client -k .
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

Download the knative manifests from 
[https://github.com/kubeflow/manifests/tree/master/knative](https://github.com/kubeflow/manifests/tree/master/knative),
I loaded all the manifest even though for this demo we only use two of the projects:
```bash
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - knative-serving-crds
  - knative-serving-install
```
The [Kubernetes Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)
whould be installed and you should be able to access it by:
```bash
kubectl proxy
```
Get token
```bash
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | awk '/^kubernetes-dashboard-token-/{print $1}') | awk '$1=="token:"{print $2}'
```
Link: [Kubernetes Dashboard](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/)

## Get kubectl access to other team memebers
See following AWS Guides:
- [AWS Doc EKS Access](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html)
- [AWS KB EKS Access](https://aws.amazon.com/premiumsupport/knowledge-center/amazon-eks-cluster-access/)

See following [configmap on other cluster for sample](https://github.com/Infoblox-CTO/athena.devops/blob/0fb8b2a4cade1ded728a2c07baecd4f16d1dde3a/cluster-provisioning/env/dev/us-west-2/eks/ml-poc/aws-auth-cm.yaml).

Start with new file from:
```bash
curl -o aws-auth-cm.yaml https://amazon-eks.s3.us-west-2.amazonaws.com/cloudformation/2020-07-23/aws-auth-cm.yaml
```

Look at current cluster configmap and use the cluster role from this step in completing the template 
```bash
kubectl describe configmap -n kube-system aws-auth
```

## Debug
### EKS Dashboard Is Empty!
It might be similar [problem on Azure AKS](https://github.com/Azure/AKS/issues/1573)?

### Deploy of knative is missing pods
The instruction called out six pods running:
```bash
NAME                                READY   STATUS    RESTARTS   AGE
activator-7746448cf9-ggk98          2/2     Running   2          18d
autoscaler-548ccfcc57-zsfpw         2/2     Running   2          18d
autoscaler-hpa-669647f4f4-mx5q7     1/1     Running   0          18d
controller-655b8c8fb8-g89x7         1/1     Running   0          18d
networking-istio-75ff868647-k95mz   1/1     Running   0          18d
webhook-5846486ff4-4ltjq            1/1     Running   0          18d
```

I only see 4 on my system, but there are 6 deployments:
```bash
❯ k -n knative-serving get pods
NAME                                READY   STATUS    RESTARTS   AGE
autoscaler-hpa-68cc87bfb9-kc8zj     1/1     Running   0          9h
controller-95dc7f8bd-ghvpf          1/1     Running   0          9h
networking-istio-5b8c5c6cff-9tgtm   1/1     Running   0          9h
webhook-67847fb4b5-z57hg            1/1     Running   0          9h
❯ k -n knative-serving get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
activator          0/1     0            0           9h
autoscaler         0/1     0            0           9h
autoscaler-hpa     1/1     1            1           9h
controller         1/1     1            1           9h
networking-istio   1/1     1            1           9h
webhook            1/1     1            1           9h
```

The activator and autoscaler both failed due to ReplicaFailure:
```bash
Conditions:
  Type             Status  Reason
  ----             ------  ------
  ReplicaFailure   True    FailedCreate
```
```bash
❯ k get rs
NAME                          DESIRED   CURRENT   READY   AGE
activator-6dc4884             1         0         0       9h
autoscaler-69bcc99c79         1         0         0       9h
autoscaler-hpa-68cc87bfb9     1         1         1       9h
controller-95dc7f8bd          1         1         1       9h
networking-istio-5b8c5c6cff   1         1         1       9h
webhook-67847fb4b5            1         1         1       9h
```

Here is the failure for activator and autoscaler ReplicaSet:
```bash
Conditions:
  Type             Status  Reason
  ----             ------  ------
  ReplicaFailure   True    FailedCreate
Events:
  Type     Reason        Age                 From                   Message
  ----     ------        ----                ----                   -------
  Warning  FailedCreate  12m (x54 over 10h)  replicaset-controller  Error creating: admission webhook "sidecar-injector.istio.io" denied the request: failed parsing generated injected YAML (check Istio sidecar injector configuration): error converting YAML to JSON: yaml: line 4: could not find expected ':'
```

I have this log in 
```bash
error: there is no need to specify a resource type as a separate argument when passing arguments in resource/name form (e.g. 'kubectl get resource/<resource_name>' instead of 'kubectl get resource resource/<resource_name>'
```

### kfserving/knative problems
Installed the Models and don't see them in knative:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubeflow/kfserving/master/docs/samples/tensorflow/tensorflow.yaml
kubectl apply -f https://raw.githubusercontent.com/kubeflow/kfserving/master/docs/samples/pytorch/pytorch.yaml
kubectl apply -f https://raw.githubusercontent.com/kubeflow/kfserving/master/docs/samples/sklearn/sklearn.yaml

kubectl get inferenceservice

kn service list
```

This is a good guide on [testing kfserving](https://github.com/kubeflow/kfserving#test-kfserving-installation)
There is also this 
[kfserving trouble-shooting guide](https://github.com/kubeflow/kfserving/blob/master/docs/DEVELOPER_GUIDE.md#troubleshooting).

Ok the problem was the kfserving is installed by default and the
instructions from [AWS to install it](https://www.kubeflow.org/docs/aws/aws-e2e/#deploy-kfserving)
was not only unnecessary they were stepping on the existing
installation, so after I deleted that install and did the
kubeflow apply the installation started to wrok properly and
I had the knative end points, in knative service list.
