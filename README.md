# local-kube-playground
A playground to try data science tools on a local Kubernetes cluster. 


## Multi-node Kubernetes Cluster
Minikube doesn't support multi-node clusters, so we can either use a hypervisor (VirtualBox, VMWare, ...) and install everything manually on each VM, or we can use [Kind](https://github.com/kubernetes-sigs/kind) (Kubernetes In Docker) to create the cluster with multiple containers acting as the nodes. Here we go with the latter appraoch (take a look at this [medium blog](https://medium.com/@raj10x/multi-node-kubernetes-cluster-with-vagrant-virtualbox-and-kubeadm-9d3eaac28b98) to create a multi-node cluster using VMs):

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
Now we can install different applications on our kubernetes cluster. Let's install Spark and its history server that provides the Web UI.

### Spark Operator
[Spark Operator](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator) is a Kubernetes operator for managing the lifecycle of Apache Spark applications on Kubernetes. You can read about Operator Pattern in more details in Kubernetes' [official website](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/). 
1. Install Spark Operator using helm:
```console
foo@bar:~$ helm install incubator/sparkoperator --namespace spark-operator --set enableWebhook=true
```
We need `--set enableWebhook=true` to set up the history server. 
Confirm the operator is installed in your cluster:
```console
foo@bar:~$  kubectl get namespaces
default           Active   30m
kube-node-lease   Active   30m
kube-public       Active   30m
kube-system       Active   30m
spark-operator    Active   15s
```
For more information look at Spark Operator's [quick start's doc](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/quick-start-guide.md). For instance, you can use the example rbac template to set up a simple service account and the RBAC for the Spark Application:
```console
foo@bar:~$ kubectl create -f  spark-on-k8s-operator/manifes/spark-rbac.yaml
```console

### Run the example pipeline
There are some examples in spark-operator's repo that you can run:
```console
foo@bar:~$ kubectl create -f  spark-on-k8s-operator/examples/spark-py-pi.yaml
```

### Spark History Server
Installing the history server is a little bit challenging. If you run one of the examples in spark-operator's repo or look at the [architecture](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/architecture-diagram.png) of the operator you realize that the pods that run the spark job have a clear lifecyle. Spark Operator creates the driver and executor pods, they run the job and on completion the die and change their status to _Completed_. When the job is completed you can't access the logs of the job nor the Web UI as they're only accessible while the job is running. 
So in order to save the logs somewhere that doesn't have the same lifecycle as the pods, we need to ensure that:
1. All the applications store event logs in a specific location (filesystem, s3, hdfs etc).
2. Deploy the history server in the cluster with access to the event logs location.

In our case we want everything to work locally, so we create an NFS server in our cluster that provides NFS Persistent Volumes which can be used & shared by other pods. Then we install the spark history server and let it read the logs from the shared location in which the spark application store the event logs (you can use any other persistent volume. There are a lot of examples online explaining how to use cloud storage options like Amazon S3 or Google Cloud Storage).

#### NFS Provisioner
Kubernetes [doesn't](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) have a built-in provisioner for NFS. So we have to set up the NFS server and add it as a provisioner. The following Chart automatically sets up the server as a StatefulSet and defines the StorageClass.
```console
foo@bar:~$ helm install stable/nfs-server-provisioner
```
Note: In Kubernetes 1.16 some api has been changed and at the time of this writing a lot of Charts have not been updated. So if you get a similar error as following, download the repo and replace `extensions/v1beta2` and `apps/v1beta2` with `apps/v1` as explained [here](https://kubernetes.io/blog/2019/09/18/kubernetes-1-16-release-announcement/). 
```bash
Error: validation failed: unable to recognize "": no matches for kind "StatefulSet" in version "apps/v1beta2"
```

#### PersitentVolumeClaim (PVC)
Create a [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims). It has to be on the same namesapce that you installed the NFS server.
```console
foo@bar:~$ kubectl create -f spark/spark-operator-pvc.yaml
```
Confirm that the PVC has a `Bound` status:
```console
foo@bar:~$  kubectl get PersistentVolumeCLaim
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
spark-pvc   Bound    pvc-0444884c-6b27-40c4-9d72-90bc905956a8   500Mi      RWX            nfs            15s
```

#### History Server Pod
Now that we have a NFS provisioner and a PVC bound to it, we can define add the claim as a volume to the history server. Before that we need to make sure the NFS server runs automatically on boot in each pod, so we have to install the `nfs-kernel-server` package on the nodes; it contains the relevant start-up scripts:
```console
foo@bar:~$ apt-get update
foo@bar:~$ apt install nfs-kernel-server
```

Install the history server:
```console
foo@bar:~$ kubectl create -f spark-operator-history-server.yaml
```
If you look at the yaml file, you see how the PVC is added as a volume and mounted to `/data` folder:
```yaml
          volumeMounts:
            - name: data
              mountPath: /data

      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: spark-pvc
          readOnly: true
```

The history server should be running and able to show all the logs that are stored in the PersistentVolume the PVC defined. The only thing left is to make the new spark jobs store their logs in the same shared volume. So make sure all your spark applications have event logging enabled and at the correct path:
```yaml
  sparkConf:
    "spark.eventLog.enabled": "true"
    "spark.eventLog.dir": "file:/data"
```

Run your pipeline and open the history server's WebUI:
```console
foo@bar:~$ kubectl create -f spark-on-k8s-operator/examples/spark-pi.yaml
sparkapplication.sparkoperator.k8s.io/spark-pi created
foo@bar:~$ kubectl get pods
NAME                                     READY   STATUS      RESTARTS   AGE
maudlin-molly-nfs-server-provisioner-0   1/1     Running     0          43m
spark-history-server-pod                 1/1     Running     0          5m13s
spark-pi-driver                          0/1     Completed   0          2m22s

foo@bar:~$ kubectl port-forward spark-history-server-pod 18080:18080
```










