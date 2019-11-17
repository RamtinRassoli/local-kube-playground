# local-kube-playground
A playground to try data science tools on a local Kubernetes cluster. 


## Multi-node Kubernetes Cluster
Minikube doesn't support multi-node clusters, so we can either use a hypervisor (VirtualBox, VMWare, ...) and install everything manually on each MV, or you can use [Kind](https://github.com/kubernetes-sigs/kind) (Kubernetes In Docker) to create the cluster with multiple containers acting as the nodes. Here we go with the latter appraoch (take a look at [medium blog](https://medium.com/@raj10x/multi-node-kubernetes-cluster-with-vagrant-virtualbox-and-kubeadm-9d3eaac28b98) to create a multi-node cluster using VMs):

1. Install [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
2. Install [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/)
3. Create the cluster (modify `kind-example-config.yaml` to sepcify your confiugations):  
```console
foo@bar:~$ kind create cluster --config kind-example-config.yaml --name my-cluster
```
4. Set the kubeconfig of the cluster (so kubectl knows how to interact with the cluster):
```console
foo@bar:~$ kind get kubeconfig > kubeconfig.yaml --name my-cluster
foo@bar:~$ export KUBECONFIG=kubeconfig.yaml 
```

## Helm
1. Install [Helm](https://helm.sh/docs/intro/install/). It makes it easier to install some of the applications and resources.
2. [Initialize](https://rancher.com/docs/rancher/v2.x/en/installation/ha/helm-init/) Helm. For Helm to work properly it needs a service account for tiller (the server-side component of Helm) with `cluster-admin` role:
```console
foo@bar:~$ kubectl -n kube-system create serviceaccount tiller

foo@bar:~$ kubectl create clusterrolebinding tiller --clusterrole=cluster-admin --serviceaccount=kube-system:tiller

foo@bar:~$ helm init --service-account tiller
```
(Don't worry about the security warning Helm gives, this is something you really need to pay attention to though if you'd like to use Helm in production. Take a look at this [article](https://engineering.bitnami.com/articles/running-helm-in-production.html) for more information)

## Applications (Spark as an example)
Now we can install different applications on our kubernetes cluster and play with them. 








