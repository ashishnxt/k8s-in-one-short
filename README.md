**core concept:**
- monolithic vs microservices
- kubernetes arthitecture
- setup on local/AWS
- kubectl
- pods
- Namespaces
- Labels
- selectors
- Annotation


![[Pasted image 20261024163322.png]]


Kubernetes Cluster: 
- Kubeadm
- MiniKube (local/EC2)
- KIND Cluster
- EKS/ AKS/ GKE

Install KIND Cluster
ref: https://github.com/LondheShubham153/kubestarter

![[Pasted image 20261024172229.png]]


```yml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
- role: control-plane
  image: kindest/node:v1.31.2
- role: worker
  image: kindest/node:v1.31.2
- role: control-plane
  image: kindest/node:v1.31.2
- role: control-plane
  image: kindest/node:v1.31.2
- role: control-plane
  image: kindest/node:v1.31.2
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
```


```bash
kind create cluster --name=k8s-cluster  --config=config.yml
```

![[Pasted image 20261024190121.png]]

```bash
kubectl get nodes
```

![[Pasted image 20261024190043.png]]




---

# Setup kubeadm on EC2:

ref: [kubestarter/Kubeadm_Installation_Scripts_and_Documentation at main · LondheShubham153/kubestarter · GitHub](https://github.com/LondheShubham153/kubestarter/tree/main/Kubeadm_Installation_Scripts_and_Documentation)

![[Pasted image 20261024202006.png]]

Add 6443 port as inbound for connect both node each other via a Kubeadm

![[Pasted image 20261024202044.png]]

![[Pasted image 20261024195917.png]]

Join worker in the Cluster:

![[Pasted image 20261024200737.png]]

![[Pasted image 20261024200824.png]]


Run a container: 

![[Pasted image 20261024201902.png]]


Namespace: 

![[Pasted image 20261024203245.png]]

```bash
kubectl get namespace 
#or
kubectl get ns

kubectl create ns <name of namespace>

#for create a nginx pod in default namespace
#command is similar like docker command to run nginx container
kubectl run nginx --image=nginx

kubectl delete pod nginx  #for delete

```




![[Pasted image 20261024203506.png]]


![[Pasted image 20261024204958.png]]


```bash
#create
kubectl create ns nginx
kubectl run nginx --image=nginx -n nginx #for run in nginx namespace
kubectl get pods -n nginx

#DELETE
kubectl delete pods nginx  -n nginx
kubectl delete ns nginx
```

![[Pasted image 20261025002437.png]]


# Creating YAML

YAML ->  Manifest file
![[Pasted image 20261025002214.png]]

namespace.yml
```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: nginx
```

![[Pasted image 20261025003146.png]]

pod.yml: 
```yml
kind: Pod
apiVersion: v1
metadata:
  name: nginx-pod
  namespace: nginx

spec:
  containers:
  -  name: nginx
     image: nginx:latest
     ports:
     - containerPort: 80
```

```yml
kubectl apply -f pod.yml
kubectl get pods -n nginx
```

> *if using AWS add port 10250 in inbound rule of Security Group

![[Pasted image 20261025005239.png]]

```bash
kubectl exec -it nginx-pod -n nginx -- bash  #for get into the container
#OR 
kubectl exec -it pod/nginx-pod -n nginx -- bash 

kubectl describe pod/nginx-pod -n nginx  # similar to docker inspect + logs
```

![[Pasted image 20261025005820.png]]

---

# Workloads: 
Deployments, StatefulSets, DaemonSets, ReplicaSets, Jobs, CronJobs.

![[Pasted image 20261025013641.png]]

All these help you to replicate the containers/pods but deployment also help in RollingUpdate (rollout)

 >All these are almost similar yml with there own specific advantage:
> 1. ReplicaSet 
> 2. StatefulSet
> 3. Deployment  rollout: while the pod update the some pods are in update state and some are in running not all pods are update in same time this is rollingUpdate

deployment.yml:
```yml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nginx-deployment
  namespace:  nginx

spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      name: nginx-dep-pod
      labels:
        app: nginx

    spec:
      containers:
      - name:  nginx
        image: nginx:latest
        ports:
        -  containerPort: 80
```

![[Pasted image 20261025012830.png]]



how to scale using deployment:

```yml
kubectl scale deployment/nginx-deployment -n nginx --replicas=5
kubectl get pods -n nginx

kubectl get pods -n nginx o wide  #for more details about pods
```
![[Pasted image 20261025013425.png]]


RollingUpdate:
```yml
kubectl set image deployment/nginx-deployment -n nginx nginx=nginx:1.27.1
```
![[Pasted image 20261025015152.png]]

![[Pasted image 20261025015252.png]]


ReplicaSet:

> *ReplicaSet and Deployment are both same and also there yml. but Deployment have rollingUpdate feature

```yml
kind: ReplicaSet
apiVersion: apps/v1
metadata:
  name: nginx-replicaset
  namespace:  nginx

spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      name: nginx-rep-pod
      labels:
        app: nginx

    spec:
      containers:
      - name:  nginx
        image: nginx:latest
        ports:
        -  containerPort: 80

```

![[Pasted image 20261025020234.png]]


DaemonSet: 

> it make sure in all available worker node have atleast one pods is running.

```yml
kind: DaemonSet
apiVersion: apps/v1
metadata:
  name: nginx-daemonset
  namespace:  nginx

spec:
  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      name: nginx-dmn-pod
      labels:
        app: nginx

    spec:
      containers:
      - name:  nginx
        image: nginx:latest
        ports:
        -  containerPort: 80

```

> *it automatically create atleast one pod according to the all available worker nodes.

![[Pasted image 20261025022737.png]]

---

Job:

>any task run for a particular period of time. it would be any thing like backup, patching, updating, etc

![[Pasted image 20261025023134.png]]

![[Pasted image 20261025024853.png]]

job.yml:
```yml
kind: Job
apiVersion: batch/v1
metadata:
  name: job
  namespace: nginx

spec:
  completions: 1
  parallelism: 1
  template:
    metadata:
      name: demo-job-pod
      labels:
        app: batch-task
    spec:
      containers:
      - name: batch-container
        image: busybox:latest
        command: ["sh", "-c", "echo Hello frinds! && sleep 10"]
      restartPolicy: Never
```


CronJob:

 >when a user want to run a job/task for a particular schedule is called a cronjob
 >*cron mean following a particular schedule. 


 
