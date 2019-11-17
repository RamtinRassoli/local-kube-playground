# local-kube-playground
A playground to try data science tools on a local Kubernetes cluster. 


## Multi-node Kubernetes Cluster
Minikube doesn't support multi-node clusters, so we can either use a hypervisor (VirtualBox, VMWare, ...) and install everything manually on each MV, or you can use [Kind](https://github.com/kubernetes-sigs/kind) (Kubernetes In Docker) to create the cluster with multiple containers acting as the nodes. Here we go with the latter appraoch (take a look at [medium blog](https://medium.com/@raj10x/multi-node-kubernetes-cluster-with-vagrant-virtualbox-and-kubeadm-9d3eaac28b98) to create a multi-node cluster using VMs).

