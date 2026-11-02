# CKA

CKA training notes.

# Architecture

- Control Plane:
  - kube-apiserver: Exposes the Kubernetes API, pass data to responsible component.
  - etcd: Stores all API server data.
  - kube-scheduler: Looks for Pods not yet bound to a node, and assigns each Pod to a suitable node.
  - kube-controller-manager: Implement Kubernetes API behavior.
  - cloud-controller-manager: Integrates with underlying cloud provider(s).
- Node:
  - Container Runtime (containerd, cri-o, etc.): Run containers.
  - kubelet: Ensures that Pods are running with Container Runtime.
  - kube-proxy: Maintains network rules on nodes to implement Service.

# Installation

## Common configuration

### Modify system settings

```bash
# enable ip forwarding
cat <<EOF | sudo tee /etc/sysctl.d/10-ipv4-forward.conf
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

# disable memory swap, also after reboot
sudo swapoff -a
sudo sed -i '/\sswap\s/ s/^/#/' /etc/fstab
```

### Install containerd

```bash
# add tools
sudo apt-get clean
sudo apt-get update
sudo apt-get install apt-transport-https ca-certificates curl gnupg lsb-release -y

# add containerd repository
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# install containerd
sudo apt-get update && sudo apt-get install containerd.io -y
```

### Configure containerd

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
grep SystemdCgroup /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl is-active containerd
```

### Install kubernetes tools

```bash
# add kubernetes repository
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# install kubernetes tools
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Controller configuration

### Init cluster

The Pod CIDR must not overlap with the node network. The CNI must use the same CIDR.

```bash
sudo kubeadm init --pod-network-cidr=172.16.0.0/16
```

### Configure easy access to cluster

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
source <(kubectl completion bash)
```

### Make kubectl completion permanent
```bash
echo "source <(kubectl completion bash)" >> $HOME/.bashrc
```

### Check cluster

```bash
kubectl get node
kubectl get pods -n kube-system
```

### Generate join token

```bash
kubeadm token create --print-join-command
```

## Worker configuration

### Join cluster

```bash
sudo kubeadm join $CONTROLLER_IP:6443 --token $JOIN_TOKEN --discovery-token-ca-cert-hash $CA_CERT_HASH
```

## CNI installation (controller)

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/calico.yaml -O
# uncomment CALICO_IPV4POOL_CIDR and set it to the --pod-network-cidr value
vim calico.yaml
kubectl apply -f calico.yaml
kubectl get pods -n kube-system | grep calico
```

# Basic text editing

It's useful to be able to act quick on CKA exam. This means exam taker needs to be fluent in using text editor like vim (recommended) or nano.

## Basic vim

```
Move around: hjkl
Insert mode: i
Exit insert mode: esc
Copy current line: yy
Paste: p
Cut current line: dd
Undo: u
Redo: Ctrl + r
Find: /
Replace all occurrences: :%s/oldword/newword/g
Start line selection: Shift + v
Start vertical (block) selection: Ctrl + v
Indent / unindent selection: > / <
Go to beginning of file: gg
Go to end of file: Shift + g
Exit: Shift + zz / esc :wq
```

### vim settings for YAML

```bash
cat <<EOF >>~/.vimrc
set expandtab tabstop=2 shiftwidth=2
EOF
```

## Shell setup for the exam

```bash
alias k=kubectl
complete -o default -F __start_kubectl k
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"

# usage
k run nginx --image=nginx:1.30.6 $do > nginx.yaml
k delete pod nginx $now
```

## Find fields fast

```bash
kubectl explain pod.spec.containers.readinessProbe
kubectl explain deployment.spec --recursive | less
```

## Get values with jsonpath

`yq` and `jq` are possibly not installed on the exam nodes. Use jsonpath.

```bash
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
kubectl get pods --sort-by=.metadata.creationTimestamp
```

# Workload & Scheduling

## Pod

- The smallest unit of workload in Kubernetes.
- Most fields can't be updated. Some can, for example container image and tolerations (add only).
- Not recommended to create it directly. Instead, create Deployment or DaemonSet.

### Create Pod

```bash
# Imperative
kubectl run nginx --image=nginx:1.30.6
kubectl delete pods nginx

# Declarative
cat <<EOF >nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.30.6
EOF

kubectl apply -f nginx.yaml
kubectl get pods
kubectl get pods -o wide
kubectl get pods nginx
kubectl delete pod nginx
```

### Review

- Create a Pod with the following specification:
  - Name: web
  - Image: httpd:2.4.69

## Deployment

- Manage a set of Pods to run an application workload, usually stateless application.
- Stateless = No state = Not storing anything = Can be moved between nodes easily.

### Create Deployment

```bash
# create manifest from scratch
cat <<EOF >kubeapp.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubeapp
  labels:
    app: kubeapp
spec:
  replicas: 5
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
      - name: kubeapp
        image: kubenesia/kubeapp:1.0.0
EOF

# easy way to generate basic manifest
kubectl create deployment kubeapp --image=kubenesia/kubeapp:1.0.0 --dry-run=client --output=yaml > kubeapp.yaml

kubectl apply -f kubeapp.yaml
kubectl get deployment
kubectl get replicaset
kubectl get pods
kubectl describe pods
kubectl get pods --show-labels
kubectl port-forward deployment/kubeapp --address 0.0.0.0 8080:8000
```

### Scale Deployment

```bash
kubectl scale deployment kubeapp --replicas=4
kubectl get deployment
kubectl get pods
```

### Describe Deployment

```bash
kubectl describe deployment kubeapp
kubectl get deployment kubeapp -o yaml
```

### Deployment rolling update

```bash
kubectl set image deployment kubeapp kubeapp=kubenesia/kubeapp:1.1.0
watch kubectl get pod -l app=kubeapp
```

### Deployment rolling update maxUnavailable parameter

```bash
cat <<EOF >kubeapp.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeapp
  name: kubeapp
spec:
  replicas: 4
  selector:
    matchLabels:
      app: kubeapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 50%
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
        - image: kubenesia/kubeapp:1.0.0
          name: kubeapp
EOF

kubectl apply -f kubeapp.yaml
kubectl get pods

kubectl set image deployment kubeapp kubeapp=kubenesia/kubeapp:1.1.0
kubectl get pods

kubectl explain deployment.spec
kubectl explain deployment.spec.strategy
kubectl explain deployment.spec.strategy.type
kubectl explain deployment.spec.strategy.rollingUpdate
```

### Deployment rollout history

```bash
kubectl rollout status deployment kubeapp
kubectl rollout history deployment kubeapp

kubectl set image deployment kubeapp kubeapp=kubenesia/kubeapp:1.2.0

kubectl rollout status deployment kubeapp
kubectl rollout history deployment kubeapp
```

### Deployment recreate strategy

```bash
cat <<EOF >kubeapp.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeapp
  name: kubeapp
spec:
  replicas: 4
  selector:
    matchLabels:
      app: kubeapp
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
        - image: kubenesia/kubeapp:1.0.0
          name: kubeapp
EOF

kubectl apply -f kubeapp.yaml
kubectl get pods

kubectl set image deployment kubeapp kubeapp=kubenesia/kubeapp:1.1.0
kubectl get pods
```

### Get container image of a Deployment

```bash
kubectl describe deployment kubeapp | grep Image
kubectl get deployment kubeapp -o jsonpath='{.spec.template.spec.containers[*].image}'
```

### Rollback Deployment

```bash
kubectl rollout undo deployment kubeapp
kubectl rollout undo deployment kubeapp --to-revision=$REVISION_NUMBER
```

### Startup Probe

Used to protect slow starting containers. Liveness and readiness probes start only after the startup probe succeeds, so the kubelet does not kill the container while it starts.

```bash
cat <<EOF >kuard.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kuard
  name: kuard
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kuard
  template:
    metadata:
      labels:
        app: kuard
    spec:
      containers:
      - image: dlneintr/kuard:0.2.0
        name: kuard-amd64
        startupProbe:
          httpGet:
            path: /ready
            port: 6000
EOF

kubectl apply -f kuard.yaml
kubectl get pods
kubectl describe pods -l app=kuard

sed -i 's/6000/8080/g' kuard.yaml
kubectl apply -f kuard.yaml
kubectl describe pods -l app=kuard
```

### Liveness Probe

Used to check the health of the container periodically. If the probe fails, the kubelet restarts the container.

```bash
cat <<EOF >kuard.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kuard
  name: kuard
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kuard
  template:
    metadata:
      labels:
        app: kuard
    spec:
      containers:
      - image: dlneintr/kuard:0.2.0
        name: kuard-amd64
        startupProbe:
          httpGet:
            path: /ready
            port: 8080
        livenessProbe:
          httpGet:
            path: /healthy
            port: 8080
EOF

kubectl apply -f kuard.yaml
kubectl get pods
kubectl describe pods -l app=kuard
```

### Readiness Probe

Used to check if the container is ready to receive traffic. If the probe fails, the Pod is removed from the Service endpoints. The container is not restarted.

```bash
cat <<EOF >kuard.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kuard
  name: kuard
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kuard
  template:
    metadata:
      labels:
        app: kuard
    spec:
      containers:
      - image: dlneintr/kuard:0.2.0
        name: kuard-amd64
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
EOF

kubectl apply -f kuard.yaml
kubectl get pods
kubectl expose deployment kuard --port=80 --target-port=8080
kubectl get endpointslices -l kubernetes.io/service-name=kuard
```

Other probe types: `tcpSocket` (port is open) and `exec` (command exits with 0).

```yaml
readinessProbe:
  tcpSocket:
    port: 8080
livenessProbe:
  exec:
    command: ["cat", "/tmp/healthy"]
```

### Review

- Try to create a Deployment with image `dlneintr/kuard:0.2.0` with 5 replicas and access port 8080 of the container.
- Try to create a Deployment with the following spec:
  - Name: `web`
  - Two images: `nginx:1.30.6` and `dlneintr/kuard:0.2.0`
  - Replica: 10
  - Startup probe to nginx
  - Liveness probe to kuard

## DaemonSet

Ensure all nodes run a copy of a Pod. Usually used for operational tools or add-ons.

### Create DaemonSet

```bash
cat <<EOF >fluentd-elasticsearch-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # these tolerations are to have the daemonset runnable on control plane nodes
      # remove them if your control plane nodes should not run pods
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v5.0.1
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      # it may be desirable to set a high priority class to ensure that a DaemonSet Pod
      # preempts running Pods
      # priorityClassName: important
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
EOF

kubectl apply -f fluentd-elasticsearch-daemonset.yaml
kubectl -n kube-system get ds fluentd-elasticsearch
kubectl -n kube-system get ds fluentd-elasticsearch -o wide
kubectl -n kube-system get pods -o wide | grep fluentd
kubectl delete -f fluentd-elasticsearch-daemonset.yaml
```

### Review

- Create a DaemonSet with image `nginx:1.30.6`, make sure it isn't scheduled on controller node.
- Create a DaemonSet with image `httpd:2.4.69`, make sure it is scheduled on all nodes including the controller.

## StatefulSet

Like Deployment, but used to deploy stateful application. Maintain a sticky identity for each Pod:

- Stable name: `mysql-0`, `mysql-1`, ...
- Stable DNS name through a headless Service: `mysql-0.mysql`.
- Stable storage: `volumeClaimTemplates` creates one PVC for each Pod. The PVC stays when the Pod is recreated.

### Install a StorageClass

`volumeClaimTemplates` needs a StorageClass for dynamic provisioning. Install local-path-provisioner and make it the default StorageClass.

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.37/deploy/local-path-storage.yaml
kubectl patch storageclass local-path -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}'
kubectl get storageclass
```

### Create StatefulSet

```bash
cat <<EOF >mysql-statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
    - port: 3306
      name: mysql
  clusterIP: None
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: public.ecr.aws/docker/library/mysql:8.4.11
        ports:
        - containerPort: 3306
          name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "P@ssw0rd"
        - name: MYSQL_DATABASE
          value: "mydb"
        - name: MYSQL_USER
          value: "user"
        - name: MYSQL_PASSWORD
          value: "P@ssw0rd"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
EOF

kubectl apply -f mysql-statefulset.yaml
kubectl get sts -o wide
kubectl get pvc
```

### Create database

```bash
kubectl exec -ti mysql-0 -- bash
mysql -h 127.0.0.1 -u root -p mydb
select version();
create table users (id int primary key, email text);
show tables;
exit
```

### Modify database version

```bash
sed -i 's/8.4.11/8.4.13/g' mysql-statefulset.yaml
kubectl apply -f mysql-statefulset.yaml
kubectl exec -ti mysql-0 -- bash
mysql -h 127.0.0.1 -u root -p mydb
select version();
show tables;
```

### Review

- Try to create a StatefulSet for postgres.
- Check [here](https://hub.docker.com/_/postgres) for information related to the container image.

## Job

Run Pods until a number of them complete successfully. Used for one-time tasks.

```bash
kubectl create job pi --image=perl:5.44 $do -- perl -Mbignum=bpi -wle 'print bpi(100)' > pi-job.yaml
vim pi-job.yaml
# set spec.completions: 3, spec.parallelism: 2, spec.backoffLimit: 2

kubectl apply -f pi-job.yaml
kubectl get jobs
kubectl get pods -l job-name=pi
kubectl logs job/pi
```

## CronJob

Create a Job on a schedule. The schedule uses cron format: minute, hour, day of month, month, day of week.

```bash
kubectl create cronjob hello --image=busybox:1.38 --schedule="*/1 * * * *" -- sh -c 'date; echo hello'
kubectl get cronjob hello
kubectl get jobs --watch

# run the CronJob now, without waiting for the schedule
kubectl create job hello-manual --from=cronjob/hello

kubectl explain cronjob.spec | grep -E 'successfulJobsHistoryLimit|failedJobsHistoryLimit|concurrencyPolicy|suspend'
kubectl delete cronjob hello
```

## ConfigMap

### Create ConfigMap

```bash
cat <<EOF >kubeapp-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubeapp
data:
  MODE: development
  COLOR: blue
  SERVICE: kubeapp
EOF

kubectl apply -f kubeapp-configmap.yaml
kubectl get cm kubeapp
kubectl get cm kubeapp -o yaml
```

### ConfigMap as env variable with key

```bash
cat <<EOF >kubeapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeapp
  name: kubeapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
      - image: public.ecr.aws/docker/library/nginx:1.30.6
        name: kubeapp
        env:
          - name: APP_MODE
            valueFrom:
              configMapKeyRef:
                name: kubeapp
                key: MODE
EOF

kubectl apply -f kubeapp-deployment.yaml
kubectl exec -ti deployment/kubeapp -- sh
printenv | grep APP_MODE
exit
```

### ConfigMap as env variable

```bash
cat <<EOF >kubeapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeapp
  name: kubeapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
      - image: public.ecr.aws/docker/library/nginx:1.30.6
        name: kubeapp
        envFrom:
          - configMapRef:
              name: kubeapp
EOF

kubectl apply -f kubeapp-deployment.yaml
kubectl exec -ti deployment/kubeapp -- sh
printenv | grep -E 'COLOR=|MODE=|SERVICE='
exit
```

### ConfigMap as file specific key

```bash
cat <<EOF >kubeapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeapp
  name: kubeapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
      - image: public.ecr.aws/docker/library/nginx:1.30.6
        name: kubeapp
        volumeMounts:
          - mountPath: /data/config
            name: config
      volumes:
        - name: config
          configMap:
            name: kubeapp
            items:
              - key: MODE
                path: mode-file-name
EOF

kubectl apply -f kubeapp-deployment.yaml
kubectl exec -ti deployment/kubeapp -- sh
ls /data/config
cat /data/config/mode-file-name
exit
```

### ConfigMap as file

```bash
cat <<EOF >kubeapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeapp
  name: kubeapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
      - image: public.ecr.aws/docker/library/nginx:1.30.6
        name: kubeapp
        volumeMounts:
          - mountPath: /data/config
            name: config
      volumes:
        - name: config
          configMap:
            name: kubeapp
EOF

kubectl apply -f kubeapp-deployment.yaml
kubectl exec -ti deployment/kubeapp -- sh
ls /data/config
cat /data/config/MODE
exit
```

## Secret

### Create Secret

```bash
kubectl create secret generic kubeapp --from-literal=MODE=development --from-literal=COLOR=blue
kubectl get secret
kubectl get secret kubeapp -o yaml
```

### Secret as env variable

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeapp
  name: kubeapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
      - image: public.ecr.aws/docker/library/nginx:1.30.6
        name: kubeapp
        envFrom:
          - secretRef:
              name: kubeapp
EOF

kubectl exec -ti deployment/kubeapp -- sh
printenv | grep -E 'COLOR=|MODE='
exit
```

### Secret as file

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeapp
  name: kubeapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
      - image: public.ecr.aws/docker/library/nginx:1.30.6
        name: kubeapp
        volumeMounts:
          - mountPath: /data/config
            name: config
      volumes:
        - name: config
          secret:
            secretName: kubeapp
EOF

kubectl exec -ti deployment/kubeapp -- sh
ls /data/config
cat /data/config/MODE
exit
```

### Review

- Create a `postgres` StatefulSet as before, but use a `Secret` to configure `POSTGRES_PASSWORD`.
- Check with `kubectl exec -ti postgres-0 -- psql -h 127.0.0.1 -U postgres`

## Node selector

- Simplest way to add a constraint for node selection by using node labels as criteria.

### Add label to node

```bash
kubectl label node worker1 disk=hdd
kubectl label node worker2 disk=ssd
```

### Show labels

```bash
kubectl label node worker1 --list
kubectl label node worker2 --list
kubectl get nodes --show-labels
```

### Pod with nodeSelector

- Pod will be scheduled to worker2.

```bash
cat <<EOF >pod-myapp-ssd.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-ssd
spec:
  containers:
  - name: nginx
    image: nginx:1.30.6
  nodeSelector:
    disk: ssd
EOF

kubectl apply -f pod-myapp-ssd.yaml
kubectl get nodes -l disk=ssd
kubectl get pods -o wide
```

- Try to remove the label from worker2 and recreate the pod, it will be stuck in "Pending" state. Add the label again after the test.

## Taints & Tolerations

- Taints: allow node to repel or "kick" a set of pods.
- Tolerations: allow pods to "bypass" taints.
- A toleration only permits a Pod to run on a tainted node. It does not attract the Pod to that node. Use nodeSelector or node affinity for that.

### Node selector + Taints + Tolerations

- Pod will be scheduled to worker1.

```bash
kubectl taint nodes worker1 role=gpu:NoSchedule
kubectl describe node worker1 | grep Taint

cat <<EOF >pod-myapp-gpu-taints-tolerations.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-gpu-taints-tolerations
spec:
  containers:
  - name: nginx
    image: nginx:1.30.6
  nodeSelector:
    disk: hdd
  tolerations:
  - key: role
    operator: Equal
    value: gpu
    effect: NoSchedule
EOF

kubectl apply -f pod-myapp-gpu-taints-tolerations.yaml
kubectl get pods -o wide
```

### Node selector + Taints without Tolerations

- Pod will be stuck in "Pending" state.

```bash
cat <<EOF >pod-myapp-gpu-without-tolerations.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-gpu-without-tolerations
spec:
  containers:
  - name: nginx
    image: nginx:1.30.6
  nodeSelector:
    disk: hdd
EOF

kubectl apply -f pod-myapp-gpu-without-tolerations.yaml
kubectl get pods -o wide
kubectl describe pod myapp-gpu-without-tolerations | grep -A 5 Events

# Remove taint
kubectl taint nodes worker1 role-
```

### Remove label

```bash
kubectl label node worker1 disk-
kubectl label node worker2 disk-
```

## Pod affinity & anti-affinity

- Used to constraint inter-pod placement.
- Use `kubectl explain deployment.spec.template.spec.affinity` to check for available fields.

### Pod anti-affinity

- Prevent pod to be scheduled alongside of specific pods.

```bash
cat <<EOF >cache.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cache
spec:
  selector:
    matchLabels:
      app: cache
  replicas: 2
  template:
    metadata:
      labels:
        app: cache
    spec:
      containers:
      - name: redis
        image: redis:8.10.3
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - cache
            topologyKey: "kubernetes.io/hostname"
EOF

kubectl apply -f cache.yaml
kubectl get pods -o wide

kubectl scale deployment cache --replicas=4
kubectl scale deployment cache --replicas=2
```

### Pod anti-affinity + affinity

- Schedule pod alongside of specific pods while also preventing it to be placed with some other pods.

```bash
cat <<EOF >webserver.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webserver
spec:
  selector:
    matchLabels:
      app: webserver
  replicas: 2
  template:
    metadata:
      labels:
        app: webserver
    spec:
      containers:
      - name: webserver
        image: nginx:1.30.6
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - cache
            topologyKey: "kubernetes.io/hostname"
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - webserver
            topologyKey: "kubernetes.io/hostname"
EOF

kubectl apply -f webserver.yaml
kubectl get pods -o wide
kubectl scale deployment webserver --replicas=3
kubectl get pods -o wide
```

### Pod anti-affinity soft rule

- Schedule the pods even if it can't find a matching node.

```bash
cat <<EOF >webserver-pref.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webserver-pref
spec:
  selector:
    matchLabels:
      app: webserver-pref
  replicas: 3
  template:
    metadata:
      labels:
        app: webserver-pref
    spec:
      containers:
      - name: web-app
        image: nginx:1.30.6
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - cache
            topologyKey: "kubernetes.io/hostname"
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - webserver-pref
              topologyKey: "kubernetes.io/hostname"
EOF

kubectl apply -f webserver-pref.yaml
kubectl get pod
kubectl scale deploy webserver-pref --replicas=4
kubectl get pod
```

## Static Pod

- Manually run a pod on a specific node, bypassing the scheduler.
- The kubelet watches the directory in `staticPodPath` of `/var/lib/kubelet/config.yaml`. kubeadm sets it to `/etc/kubernetes/manifests`.
- The Pod name gets the node name as suffix, for example `pod-static-worker1`.
- You can't delete a static pod with kubectl. Remove the manifest file instead.

```bash
ssh $WORKER_IP
sudo grep staticPodPath /var/lib/kubelet/config.yaml
cat <<EOF | sudo tee /etc/kubernetes/manifests/pod-static.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-static
spec:
  containers:
  - name: nginx
    image: nginx:1.30.6
EOF

ls /etc/kubernetes/manifests/

ssh $CONTROLLER
kubectl get pods -o wide
```

# Resource requests and limits

- Used to control the resource usage (CPU and memory) of a Pod.
- Best practice: do not use CPU limit to prevent throttling: (https://home.robusta.dev/blog/stop-using-cpu-limits)
  - In the exam, set all requests and limits that the task asks for.
- Always define resource requests
  - Easier capacity planning
  - Less surprise

## Create Pod with resource requests and limits

```bash
cat <<EOF >pod-requests-limits.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubeapp-requests-limits
spec:
  containers:
  - name: kubeapp
    image: kubenesia/kubeapp:1.2.0
    resources:
      requests:
        memory: 128Mi
        cpu: 250m
      limits:
        memory: 256Mi
EOF

kubectl apply -f pod-requests-limits.yaml

kubectl get pods
kubectl describe pods

kubectl describe node worker1
kubectl describe node worker2
```

## Create Pod with bad resource requests and limits

- If resource requests can't be fulfilled by any node, then Pod will stuck in Pending state.

```bash
cat <<EOF >pod-requests-limits.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubeapp-requests-limits
spec:
  containers:
  - name: kubeapp
    image: kubenesia/kubeapp:1.2.0
    resources:
      requests:
        memory: 128Gi
        cpu: 250m
      limits:
        memory: 256Gi
        cpu: 500m
EOF

kubectl apply -f pod-requests-limits.yaml

kubectl get pods
kubectl describe pods

kubectl describe node worker1
kubectl describe node worker2

kubectl delete pod kubeapp-requests-limits
```

## Create ResourceQuota

Control resources per namespace instead of per pod.

```bash
kubectl create ns payment

cat <<EOF >resourcequota-payment.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: payment
  namespace: payment
spec:
  hard:
    requests.cpu: "2"
EOF
kubectl apply -f resourcequota-payment.yaml

kubectl get resourcequota -n payment

cat <<EOF >nginx-payment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: payment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.30.6
          resources:
            requests:
              cpu: 1000m
EOF
kubectl apply -f nginx-payment.yaml

kubectl get pods -n payment
```

## Priority Class

```bash
cat <<EOF >high-priority-priorityclass.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."
EOF
kubectl apply -f high-priority-priorityclass.yaml

cat <<EOF >nginx-priorityclass.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx:1.30.6
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority
EOF
kubectl apply -f nginx-priorityclass.yaml

kubectl get priorityclass
```

# Autoscaling

- Change replica number of a Deployment based on resource usage

## Install metrics-server

```bash
curl -fsSLo metrics-server.yaml https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.9.1/components.yaml
vim metrics-server.yaml
# add --kubelet-insecure-tls under args
kubectl apply -f metrics-server.yaml
kubectl -n kube-system get pods
kubectl top nodes
kubectl top pods -A
kubectl describe nodes $WORKER_NODE
```

## Create Deployment + HPA

```bash
kubectl delete pods --all
kubectl delete deployment --all

cat <<EOF >kubeapp.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: kubeapp
spec:
  type: NodePort
  selector:
    app: kubeapp
  ports:
    - port: 8000
      nodePort: 30088
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubeapp
  labels:
    app: kubeapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
      - name: kubeapp
        image: kubenesia/kubeapp:1.2.0
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
EOF

kubectl apply -f kubeapp.yaml
kubectl autoscale deployment kubeapp --min=2 --max=10 --cpu="50%" --dry-run=client --output=yaml > kubeapp-hpa.yaml
kubectl apply -f kubeapp-hpa.yaml
kubectl get hpa
```

## Load test

```bash
sudo apt install apache2-utils -y
watch kubectl get hpa
ab -n 100000 -c 100 http://localhost:30088/
```

# Services & Networking

- Containers inside the same pod can communicate via `localhost`.
- Each pod in a cluster gets its own unique IP address.
- All pods can communicate with all other pods by default.

## Service

- Service provides a stable IP address or hostname instead of manually using IP of pods.

### ClusterIP

```bash
cat <<EOF >kubeapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubeapp
  labels:
    app: kubeapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
        - name: kubeapp
          image: kubenesia/kubeapp:1.2.0
          ports:
            - name: http
              containerPort: 8000
EOF

cat <<EOF >kubeapp-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: kubeapp
  labels:
    app: kubeapp
spec:
  type: ClusterIP
  selector:
    app: kubeapp
  ports:
    - port: 80
      protocol: TCP
      targetPort: 8000
EOF

kubectl apply -f kubeapp-deployment.yaml -f kubeapp-service.yaml
kubectl get svc
kubectl describe svc kubeapp
kubectl get endpointslices -l kubernetes.io/service-name=kubeapp
kubectl describe endpointslices -l kubernetes.io/service-name=kubeapp
kubectl get pods -o wide
kubectl get pods -l app=kubeapp

kubectl run test -ti --rm --image=kubenesia/kubebox -- sh
curl kubeapp
nslookup kubeapp
exit
```

### ExternalName

```bash
cat <<EOF >gogolele-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: gogolele
spec:
  type: ExternalName
  externalName: google.com
EOF

kubectl apply -f gogolele-service.yaml
kubectl get svc
kubectl describe svc gogolele

kubectl run test -it --rm --image=kubenesia/kubebox -- sh
curl gogolele
nslookup gogolele
exit
```

### NodePort

```bash
cat <<EOF >kubeapp-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: kubeapp
spec:
  type: NodePort
  selector:
    app: kubeapp
  ports:
    - port: 80
      protocol: TCP
      targetPort: 8000
      nodePort: 31080
EOF

kubectl apply -f kubeapp-service.yaml
kubectl get svc
kubectl describe svc kubeapp

export WORKER_IP=CHANGE_ME
export NODE_PORT=$(kubectl get svc kubeapp -o jsonpath='{.spec.ports[0].nodePort}')
curl "$WORKER_IP:$NODE_PORT"
```

### Headless Service

- Used to bypass the cluster-wide IP address, name will be resolved directly to pod IPs.

```bash
cat <<EOF >kubeapp-headless-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: kubeapp-headless
spec:
  type: ClusterIP
  clusterIP: None
  selector:
    app: kubeapp
  ports:
    - port: 80
      protocol: TCP
      targetPort: 8000
EOF

kubectl apply -f kubeapp-headless-service.yaml
kubectl get svc | grep kubeapp
kubectl describe svc kubeapp
kubectl describe svc kubeapp-headless

kubectl run test -it --rm --image=kubenesia/kubebox -- sh

nslookup kubeapp
nslookup kubeapp-headless

curl kubeapp
curl kubeapp-headless:8000
```

### Review

- Run the following commands:

```bash
cat <<EOF >nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    application: nginx
    role: web
  ports:
    - port: 80
      protocol: TCP
      targetPort: 8080
EOF
kubectl apply -f nginx-service.yaml
kubectl create deployment nginx --image=nginx:1.30.6
kubectl port-forward svc/nginx --address 0.0.0.0 1234:8080
curl localhost:1234
```

- Does it work? If not, what's wrong?
- Create a Deployment `echo` with image `mendhak/http-https-echo:43`. Create a Service with type `NodePort` to expose the previous Deployment with the same name. The application port is `8080`. Also create a HPA with target CPU utilization of 60% and max replicas of 5.

## Ingress

- Route HTTP/HTTPS traffic into cluster workloads.
- The ingress-nginx project is retired and gets no more updates. Use Gateway API for new work. The exam can still ask for Ingress.

### Install ingress controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

### Create Deployment & Service

```bash
kubectl create deployment blue --image=dlneintr/kuard:0.2.0 --port=8080
kubectl create deployment green --image=dlneintr/kuard:0.2.0 --port=8080

kubectl expose deployment blue --port=80 --target-port=8080
kubectl expose deployment green --port=80 --target-port=8080
```

### Route by host header

```bash
kubectl create ingress blue --class=nginx --rule="blue.$WORKER_IP.sslip.io/*=blue:80"
kubectl create ingress green --class=nginx --rule="green.$WORKER_IP.sslip.io/*=green:80"

kubectl get ingress
kubectl get ingress blue -o yaml
kubectl get ingress green -o yaml

export NODE_PORT=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.spec.ports[0].nodePort}')
echo $NODE_PORT

curl http://blue.$WORKER_IP.sslip.io:$NODE_PORT
curl http://green.$WORKER_IP.sslip.io:$NODE_PORT
```

### Route by path

```bash
kubectl delete deployment blue
kubectl delete deployment green
kubectl delete svc blue
kubectl delete svc green

kubectl create deployment blue --image=mendhak/http-https-echo:43
kubectl create deployment green --image=mendhak/http-https-echo:43

kubectl expose deployment blue --port=80 --target-port=8080
kubectl expose deployment green --port=80 --target-port=8080

cat <<EOF >blue-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: blue
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - backend:
          service:
            name: blue
            port:
              number: 80
        path: /blue
        pathType: Prefix
EOF

kubectl apply -f blue-ingress.yaml

cat <<EOF >green-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: green
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - backend:
          service:
            name: green
            port:
              number: 80
        path: /green
        pathType: Prefix
EOF

kubectl apply -f green-ingress.yaml

curl http://$WORKER_IP.sslip.io:$NODE_PORT/blue
curl http://$WORKER_IP.sslip.io:$NODE_PORT/green
```

### Configure HTTPS

```bash
# Start from clean state
kubectl delete ingress blue green

# Set IP to primary IP of the machine
export IP=$(ip -4 route get 1.1.1.1 | awk '{for (i = 1; i < NF; i++) if ($i == "src") print $(i + 1)}')

# Create self-signed certificate
openssl req -x509 -nodes -days 365 -keyout tls.key -out tls.crt -subj "/CN=*.$IP.sslip.io"
kubectl create secret tls sslip-io --key tls.key --cert tls.crt

kubectl create ingress blue --class nginx --rule=blue.$IP.sslip.io/*=blue:80,tls=sslip-io
kubectl create ingress green --class nginx --rule=green.$IP.sslip.io/*=green:80,tls=sslip-io

# Find NodePort for https
export HTTPS_NODE_PORT=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.spec.ports[1].nodePort}')
curl --insecure https://blue.$IP.sslip.io:$HTTPS_NODE_PORT
```

### Review

- Create Deployment `kubeapp1`.
  - Use image: `kubenesia/kubeapp:1.1.0`.
  - Minimum CPU is 0.5 core.
  - Minimum memory is 128 MiB.
  - Scale based on CPU utilization with maximum replicas of 10.
  - Make it accessible on `kubeapp1.$WORKER_IP.sslip.io`.

- Create Deployment `kubeapp2`.
  - Use image: `kubenesia/kubeapp:1.2.0`.
  - Minimum CPU is 1/4 core.
  - Minimum memory is 64 MiB.
  - Scale based on CPU utilization with maximum replicas of 2.
  - Make it accessible on `kubeapp2.$WORKER_IP.sslip.io`.

- Run these commands. Does it work? If not, what's wrong?

```bash
export WORKER1_IP=CHANGE_ME
cat <<EOF >echo.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
  labels:
    app: echo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - name: echo
        image: mendhak/http-https-echo:43
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: echo
spec:
  type: ClusterIP
  selector:
    app: echo
  ports:
  - port: 8080
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo
spec:
  ingressClassName: ingress-nginx
  rules:
  - host: echo.$WORKER1_IP.sslip.io
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: echo
              port:
                number: 8080
EOF
kubectl apply -f echo.yaml
export NGINX_PORT=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.spec.ports[0].nodePort}')
curl "http://echo.$WORKER1_IP.sslip.io:$NGINX_PORT"
```

## Gateway

- Gateway API is an official Kubernetes project focused on L4 and L7 routing in Kubernetes.
- Intended to be the next generation of Kubernetes Ingress, Load Balancing, and Service Mesh APIs.
- Designed to be generic, expressive, and role-oriented.

nginx-fabric-gateway will be used here as Gateway API implementation. For other implementations, check here https://gateway-api.sigs.k8s.io/implementations/.

### Install Gateway CRD

Use the Gateway API version that the controller supports. nginx-gateway-fabric 2.8.0 supports v1.6.3.

```bash
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.3/standard-install.yaml
```

### Install helm
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4 | bash
```

### Install Gateway controller implementation

```bash
cat <<EOF >/tmp/nginx-gateway.yaml
nginx:
  kind: daemonSet
  service:
    type: NodePort
    nodePorts:
      - port: 30080
        listenerPort: 80
EOF

helm upgrade --install nginx-gateway oci://ghcr.io/nginx/charts/nginx-gateway-fabric --version 2.8.0 --create-namespace -n nginx-gateway -f /tmp/nginx-gateway.yaml
```

### Create Gateway

```bash
kubectl apply -f- <<EOF
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: nginx-gateway
  namespace: nginx-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - protocol: HTTP
    port: 80
    name: http
    allowedRoutes:
      namespaces:
        from: All
EOF
```

### Route by host header

```bash
kubectl create deployment blue --image=mendhak/http-https-echo:43
kubectl create deployment green --image=mendhak/http-https-echo:43

kubectl expose deployment blue --port=80 --target-port=8080
kubectl expose deployment green --port=80 --target-port=8080

kubectl apply -f - <<EOF
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: blue
  labels:
    app: blue
spec:
  hostnames:
  - blue.example.com
  parentRefs:
  - name: nginx-gateway
    namespace: nginx-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - kind: Service
      name: blue
      port: 80
EOF

kubectl apply -f - <<EOF
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: green
  labels:
    app: green
spec:
  hostnames:
  - green.example.com
  parentRefs:
  - name: nginx-gateway
    namespace: nginx-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - kind: Service
      name: green
      port: 80
EOF

curl --connect-to ::$WORKER_IP:30080 blue.example.com
curl --connect-to ::$WORKER_IP:30080 green.example.com
```

### Route by path

```bash
kubectl apply -f - <<EOF
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: blue
  labels:
    app: blue
spec:
  parentRefs:
  - name: nginx-gateway
    namespace: nginx-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /blue
    backendRefs:
    - kind: Service
      name: blue
      port: 80
EOF

kubectl apply -f - <<EOF
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: green
  labels:
    app: green
spec:
  parentRefs:
  - name: nginx-gateway
    namespace: nginx-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /green
    backendRefs:
    - kind: Service
      name: green
      port: 80
EOF

curl --connect-to ::$WORKER_IP:30080 example.com/blue
curl --connect-to ::$WORKER_IP:30080 example.com/green
```

### Weight

```bash
kubectl apply -f - <<EOF
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: weight
spec:
  parentRefs:
  - name: nginx-gateway
    namespace: nginx-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /weight
    backendRefs:
    - kind: Service
      name: blue
      port: 80
      weight: 50
    - kind: Service
      name: green
      port: 80
      weight: 50
EOF

curl --connect-to ::$WORKER_IP:30080 example.com/weight
curl --connect-to ::$WORKER_IP:30080 example.com/weight
curl --connect-to ::$WORKER_IP:30080 example.com/weight
curl --connect-to ::$WORKER_IP:30080 example.com/weight
```

### Review

- Create two deployments `blue` and `green` with image `mendhak/http-https-echo:43`.
- The deployment should be accessible on `loadbalance.com` with 80:20 balancing between blue and green. Use HTTPRoute with name `loadbalance` to achieve this.
- Run `curl -s --connect-to ::$WORKER_IP:30180 loadbalance.com | grep hostname` multiple times to check the result.

## Network Policy

Limit network connection between pods.

### Deny from other namespace

```bash
# Create target workload
kubectl create deployment kubeapp --image=kubenesia/kubeapp:1.2.0 --port=8000
kubectl expose deployment kubeapp --port=80 --target-port=8000

# Create NetworkPolicy to prevent access from other namespace
cat <<EOF >netpol-deny-from-other-ns.yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: default
  name: deny-from-other-namespaces
spec:
  podSelector: {}
  ingress:
  - from:
    - podSelector: {}
EOF
kubectl apply -f netpol-deny-from-other-ns.yaml

# Check NetworkPolicy
kubectl get networkpolicy

# Test access from other namespace
kubectl create ns dev
kubectl run test -it -n dev --rm --image=kubenesia/kubebox -- sh
wget -qO- --timeout=2 kubeapp.default # fail
kubectl run test -it --rm --image=kubenesia/kubebox -- sh
wget -qO- --timeout=2 kubeapp # ok

# Remove NetworkPolicy
kubectl delete netpol deny-from-other-namespaces
```

### Allow only from pod with specific labels
```bash
kubectl create ns marketplace

kubectl create deployment backend --namespace=marketplace --image=nginx:1.30.6
kubectl create deployment frontend --namespace=marketplace --image=nginx:1.30.6
kubectl create deployment experiment --namespace=marketplace --image=nginx:1.30.6

kubectl expose deployment backend --namespace=marketplace --port=80
kubectl expose deployment frontend --namespace=marketplace --port=80
kubectl expose deployment experiment --namespace=marketplace --port=80

cat <<EOF >marketplace-netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-from-frontend
  namespace: marketplace
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
EOF
kubectl apply -f marketplace-netpol.yaml

kubectl exec -n marketplace -ti deployment/frontend -- bash
curl -m 5 backend.marketplace # ok

kubectl exec -n marketplace -ti deployment/experiment -- bash
curl -m 5 backend.marketplace # fail

kubectl delete -f marketplace-netpol.yaml
```

### Allow only from specific namespace

```bash
cat <<EOF >allow-from-specific-namespace-netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-from-research
  namespace: marketplace
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          department: research
EOF
kubectl apply -f allow-from-specific-namespace-netpol.yaml

kubectl create namespace foobar
kubectl label namespace foobar department=research

kubectl run test -it -n foobar --rm --image=kubenesia/kubebox -- sh
curl -m 5 backend.marketplace # ok

kubectl run test -it -n default --rm --image=kubenesia/kubebox -- sh
curl -m 5 backend.marketplace # fail
```

### Review

- Create new namespaces `ride` and `food`.
- Inside namespace `ride`, create 3 deployments:
  - `gateway`
  - `travel`
  - `shadow`
- Inside namespace `food`, create 3 deployments:
  - `order`
  - `tenant`
  - `promo`
- Create NetworkPolicy `allow-from-ride` on namespace `food` to allow access from namespace `ride`.
- Create NetworkPolicy `allow-gateway-from-travel` on namespace `ride` to only allow pod with label `app=travel` to access pod with label `app=gateway` inside the same namespace.
- All deployments should use image `nginx:1.30.6`.
- All deployments should have a Service with type `ClusterIP`.

# Storage

## Share directory between containers inside a pod using emptyDir

```bash
cat <<EOF >pod-with-empty-dir.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-empty-dir
spec:
  containers:
  - name: nginx
    image: nginx:1.30.6
    volumeMounts:
    - name: wwwroot
      mountPath: /usr/share/nginx/html
  - name: caddy
    image: caddy:2.11.7
    command:
      - caddy
      - file-server
      - --root
      - /var/www/html
      - --listen
      - :8080
    volumeMounts:
    - name: wwwroot
      mountPath: /var/www/html
  volumes:
  - name: wwwroot
    emptyDir:
      sizeLimit: 500Mi
EOF

kubectl apply -f pod-with-empty-dir.yaml

# port-forward blocks the terminal. Run curl in a second terminal, then stop port-forward with Ctrl + c.
kubectl exec -ti -c nginx pod-with-empty-dir -- bash
echo "Test emptyDir volume with containers" >/usr/share/nginx/html/index.html
exit

kubectl port-forward pod-with-empty-dir 1234:80
curl -v localhost:1234

kubectl port-forward pod-with-empty-dir 1234:8080
curl -v localhost:1234

kubectl exec -ti -c caddy pod-with-empty-dir -- sh
echo "New page" >/var/www/html/index.html
exit

kubectl port-forward pod-with-empty-dir 1234:80
curl -v localhost:1234

kubectl port-forward pod-with-empty-dir 1234:8080
curl -v localhost:1234
```

## Init container and sidecar container

- Init container: runs to completion before the app containers start.
- Sidecar container: an init container with `restartPolicy: Always`. It starts before the app containers and runs for the full life of the Pod.

```bash
cat <<EOF >pod-with-sidecar.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-sidecar
spec:
  initContainers:
  - name: init
    image: busybox:1.38
    command: ["sh", "-c", "echo init done > /var/log/app/init.log"]
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  - name: log-shipper
    image: busybox:1.38
    restartPolicy: Always
    command: ["sh", "-c", "tail -F /var/log/app/app.log"]
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  containers:
  - name: app
    image: busybox:1.38
    command: ["sh", "-c", "while true; do date >> /var/log/app/app.log; sleep 5; done"]
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
  volumes:
  - name: logs
    emptyDir: {}
EOF

kubectl apply -f pod-with-sidecar.yaml
kubectl get pod pod-with-sidecar
kubectl logs pod-with-sidecar -c log-shipper -f
```

To add a sidecar to an existing Deployment, edit `spec.template.spec` with `kubectl edit deployment $NAME`. Add the shared volume, the `volumeMounts` in both containers, and the sidecar container.

## Set up NFS server

```bash
# Controller
sudo apt-get install --yes nfs-kernel-server
sudo mkdir -p /srv/nfs4
sudo chown 65534:65534 /srv/nfs4
# change 10.0.0.0/8 to the network of your nodes
cat <<EOF | sudo tee /etc/exports
/srv/nfs4 10.0.0.0/8(rw,no_subtree_check,all_squash)
EOF
sudo systemctl restart nfs-kernel-server
sudo exportfs -v

# Worker
sudo apt-get install --yes nfs-common
```

## Static provisioning

```bash
export NFS_SERVER=$(ip -4 a | grep global | head -n 1 | awk '{print $2}' | cut -d '/' -f 1)

cat <<EOF >nginx-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx
spec:
  storageClassName: "nfs"
  capacity:
    storage: 1Gi
  accessModes:
   - ReadWriteMany
  nfs:
    server: $NFS_SERVER
    path: /srv/nfs4
EOF

kubectl apply -f nginx-pv.yaml

kubectl get pv
kubectl describe pv nginx

cat <<EOF >nginx-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx
spec:
  storageClassName: "nfs"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
EOF

kubectl apply -f nginx-pvc.yaml
kubectl get pvc
kubectl describe pvc nginx
```

## Mount volume on Pod

```bash
cat <<EOF >nginx-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.30.6
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
  volumes:
  - name: html
    persistentVolumeClaim:
      claimName: nginx
EOF

kubectl apply -f nginx-pod.yaml
kubectl describe pods nginx
kubectl get pods nginx -o wide

kubectl port-forward nginx 8080:80

echo "Old landing page" | sudo tee /srv/nfs4/index.html
curl localhost:8080

echo "New landing page" | sudo tee /srv/nfs4/index.html
curl localhost:8080

kubectl delete pods nginx
kubectl delete pvc nginx
kubectl delete pv nginx
```

## Match a PVC to an existing PV

A PVC binds to a PV only when all of these match:

- `storageClassName` is the same. Use `""` to bind to a PV without a StorageClass.
- `accessModes` of the PV include the requested mode.
- Capacity of the PV is equal to or larger than the request.
- `volumeMode` is the same (default `Filesystem`).

```bash
kubectl get pv -o custom-columns=NAME:.metadata.name,CLASS:.spec.storageClassName,MODES:.spec.accessModes,SIZE:.spec.capacity.storage,STATUS:.status.phase
```

## Dynamic provisioning with StorageClass

- A StorageClass tells the provisioner how to create a PV when a PVC asks for one.
- `reclaimPolicy`: `Delete` (default) removes the PV with the PVC. `Retain` keeps the PV and data.
- `volumeBindingMode`: `Immediate` (default) creates the PV at once. `WaitForFirstConsumer` waits for a Pod, so the PV is created on the node of the Pod.
- This example uses the local-path provisioner from the StatefulSet section.

```bash
cat <<EOF >local-retain-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-retain
provisioner: rancher.io/local-path
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: false
EOF
kubectl apply -f local-retain-storageclass.yaml

cat <<EOF >data-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  storageClassName: local-retain
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
kubectl apply -f data-pvc.yaml

# Pending until a Pod uses it, because of WaitForFirstConsumer
kubectl get pvc data

kubectl run data --image=nginx:1.30.6 --overrides='{"spec": {"volumes": [{"name": "data", "persistentVolumeClaim": {"claimName": "data"}}], "containers": [{"name": "data", "image": "nginx:1.30.6", "volumeMounts": [{"name": "data", "mountPath": "/data"}]}]}}'
kubectl get pvc data
kubectl get pv

# make a StorageClass the default
kubectl patch storageclass local-retain -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}'
```

# Security

## Create additional user

### Create certificate

```bash
# Create Certificate Signing Request
openssl req -nodes -new -keyout andy.key -out andy.csr -subj "/CN=andy/O=dev"

# Sign the CSR using Kubernetes CA
sudo openssl x509 -req -in andy.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out andy.crt -days 365

# View the signed certificate content
openssl x509 -in andy.crt -text -noout
```

### Sign with CertificateSigningRequest API

Alternative to signing with `ca.key`. Use this when you have no access to the CA key.

```bash
cat <<EOF >andy-csr.yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: andy
spec:
  request: $(base64 -w 0 andy.csr)
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages:
  - client auth
EOF
kubectl apply -f andy-csr.yaml

kubectl get csr
kubectl certificate approve andy
kubectl get csr andy -o jsonpath='{.status.certificate}' | base64 -d > andy.crt
```

### Create new kubeconfig context

```bash
# Create new context for the new user
kubectl config set-credentials andy --client-certificate=andy.crt --client-key=andy.key --embed-certs=true
kubectl config set-context andy@kubernetes --cluster=kubernetes --user=andy

# Change context
kubectl config get-contexts
kubectl config use-context andy@kubernetes

# Both commands will fail
kubectl get nodes
kubectl get pods

# View content of kubeconfig
cat ~/.kube/config
```

### Create ClusterRole

```bash
kubectl config use-context kubernetes-admin@kubernetes

cat <<EOF >andy-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: andy
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - services
    verbs:
      - get
      - list
      - watch
EOF

kubectl apply -f andy-clusterrole.yaml
```

### Create ClusterRoleBinding

```bash
cat <<EOF >andy-clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: andy
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: andy
subjects:
  - kind: User
    name: andy
EOF

kubectl apply -f andy-clusterrolebinding.yaml
```

## Test RBAC (ClusterRole)

```bash
kubectl config use-context andy@kubernetes

# should success
kubectl get pods -n kube-system
kubectl get svc -n kube-system

# should fail
kubectl get secret -n kube-system

kubectl auth can-i get pods
kubectl auth can-i get svc

kubectl auth can-i get secrets

# back to admin
kubectl config use-context kubernetes-admin@kubernetes

# delete ClusterRole and ClusterRoleBinding
kubectl delete clusterrolebinding andy
kubectl delete clusterrole andy
```

## Create Role

```bash
cat <<EOF >andy-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: andy
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - services
    verbs:
      - get
      - list
      - watch
EOF

kubectl apply -f andy-role.yaml
```

## Create RoleBinding

```bash
cat <<EOF >andy-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: andy
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: andy
subjects:
  - kind: User
    name: andy
EOF

kubectl apply -f andy-rolebinding.yaml
```

## Test RBAC (Role)

```bash
kubectl config use-context andy@kubernetes

# should success
kubectl get pods
kubectl get svc

# should fail
kubectl get pods -n kube-system
kubectl get svc -n kube-system

# should fail
kubectl get secret

kubectl auth can-i get pods
kubectl auth can-i get svc

kubectl auth can-i get secrets

# back to admin
kubectl config use-context kubernetes-admin@kubernetes

# delete Role and RoleBinding
kubectl delete rolebinding andy
kubectl delete role andy
```

## ServiceAccount

- Identity for processes that run in a Pod.
- Bind a Role or ClusterRole to a ServiceAccount with `--serviceaccount=$NAMESPACE:$NAME`.

```bash
kubectl create serviceaccount pod-reader
kubectl create role pod-reader --verb=get,list,watch --resource=pods
kubectl create rolebinding pod-reader --role=pod-reader --serviceaccount=default:pod-reader

# check permission without switching context
kubectl auth can-i list pods --as=system:serviceaccount:default:pod-reader # yes
kubectl auth can-i list secrets --as=system:serviceaccount:default:pod-reader # no

# use a token of the ServiceAccount
export TOKEN=$(kubectl create token pod-reader)
kubectl get pods --token=$TOKEN # ok
kubectl get secrets --token=$TOKEN # fail

# use the ServiceAccount in a Pod
kubectl run nginx --image=nginx:1.30.6 --overrides='{"spec": {"serviceAccountName": "pod-reader"}}'
kubectl get pod nginx -o jsonpath='{.spec.serviceAccountName}'

kubectl delete pod nginx
kubectl delete rolebinding pod-reader
kubectl delete role pod-reader
kubectl delete serviceaccount pod-reader
```

## Review

- Create a ClusterRole and ClusterRoleBinding with name `scofield` to allow user `scofield` to:
  - get, list, create, and delete ingress.
  - get and list all pods.

# Troubleshooting

## Debugging Pod

```bash
# list pods
kubectl get pods -o wide

# execute command on running pods
kubectl exec $POD_NAME -- $COMMAND

# interactive command
kubectl exec -it $POD_NAME -- sh

# get details of pod
kubectl describe pod $POD_NAME
kubectl get pod $POD_NAME -o yaml

# show logs of pod
kubectl logs $POD_NAME
kubectl logs $POD_NAME -f
kubectl logs $POD_NAME -c $CONTAINER_NAME

# show logs of the previous container, after a crash
kubectl logs $POD_NAME --previous

# show events, newest last
kubectl get events --sort-by=.lastTimestamp
```

## Debugging Service

```bash
# list services
kubectl get svc

# get details of service
kubectl describe svc $SVC_NAME

# get list of pods based on labels
kubectl get pods -l $KEY=$VALUE

# no endpoints means the selector does not match any ready pod
kubectl get endpointslices -l kubernetes.io/service-name=$SVC_NAME
```

## Debugging cluster

```bash
# list nodes
kubectl get node

# show cluster status
kubectl cluster-info

# check log of control plane components
kubectl logs -n kube-system -l=component=kube-apiserver
kubectl logs -n kube-system -l=component=kube-scheduler
kubectl logs -n kube-system -l=component=kube-controller-manager
kubectl logs -n kube-system -l=component=etcd
```

## Debugging node

Use these commands on the node. They also work when kube-apiserver is down and `kubectl` fails.

```bash
# kubelet status and logs
sudo systemctl status kubelet
sudo journalctl -u kubelet --no-pager | tail -n 50

# containerd status
sudo systemctl status containerd

# containers on the node, also stopped ones
sudo crictl ps -a
sudo crictl logs $CONTAINER_ID

# log files of control plane pods
sudo ls /var/log/pods/
sudo tail -n 50 /var/log/pods/kube-system_kube-apiserver-*/kube-apiserver/*.log

# static pod manifests of control plane components
sudo ls /etc/kubernetes/manifests/

# kubelet configuration
sudo cat /var/lib/kubelet/config.yaml
```

Common causes:

- Node `NotReady`: kubelet or containerd is stopped, swap is on, or the CNI is not installed.
- kube-apiserver down: wrong flag, path, or certificate in `/etc/kubernetes/manifests/kube-apiserver.yaml`, or etcd is not reachable.
- Pod `Pending`: no node has enough resources, a taint has no toleration, or a PVC is not bound. Check `kubectl describe pod`.

# Cluster maintenance

## etcd backup

```bash
# backup existing etcd manifests
sudo cp /etc/kubernetes/manifests/etcd.yaml etcd.yaml

# install client tools
sudo apt-get install etcd-client

# find the certificate paths and the endpoint
sudo grep -E 'cert-file|key-file|trusted-ca-file|listen-client-urls' /etc/kubernetes/manifests/etcd.yaml

# create ns to verify restore later
kubectl create ns before-backup

# do backup
sudo ETCDCTL_API=3 etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    snapshot save snapshot.db

kubectl delete ns before-backup

# view backup result
# in etcd 3.5 and later, `etcdctl snapshot status` is deprecated. Use `etcdutl` if it is installed.
sudo ETCDCTL_API=3 etcdctl --write-out=table snapshot status snapshot.db
sudo etcdutl --write-out=table snapshot status snapshot.db
```

## etcd restore

```bash
# in etcd 3.5 and later, `etcdctl snapshot restore` is deprecated. Use `etcdutl` if it is installed.
sudo ETCDCTL_API=3 etcdctl \
    --data-dir /var/lib/etcd-from-backup \
    snapshot restore snapshot.db
sudo etcdutl --data-dir /var/lib/etcd-from-backup snapshot restore snapshot.db

sudo vim /etc/kubernetes/manifests/etcd.yaml
# replace /var/lib/etcd to /var/lib/etcd-from-backup on hostPath

# wait until the kubelet restarts etcd and kube-apiserver
sudo crictl ps | grep -E 'etcd|kube-apiserver'

# ns should be restored
kubectl get ns before-backup
```

## Version upgrade

- Upgrade one minor version at a time, for example 1.36 to 1.37.
- Upgrade the controller first, then the workers.
- Upgrade kubeadm first. Upgrade kubelet and kubectl after `kubeadm upgrade`.
- The packages are on hold (see installation), so remove the hold before the upgrade and set it again after.

### Controller

```bash
kubectl get nodes -o wide

# upgrade kubeadm
sudo sed -i 's/v1.36/v1.37/g' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update && apt-cache madison kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm='1.37.*'
sudo apt-mark hold kubeadm
kubeadm version -o short

# upgrade control plane components
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply $(kubeadm version -o short)

# upgrade kubelet and kubectl
kubectl drain $CONTROLLER_NODE --ignore-daemonsets
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet='1.37.*' kubectl='1.37.*'
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl uncordon $CONTROLLER_NODE

kubectl get nodes -o wide
```

### Worker

```bash
ssh $WORKER_IP

# upgrade kubeadm
sudo sed -i 's/v1.36/v1.37/g' /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update && apt-cache madison kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm='1.37.*'
sudo apt-mark hold kubeadm

# upgrade kubelet configuration
sudo kubeadm upgrade node

# drain the worker from the controller
ssh $CONTROLLER
kubectl drain $WORKER_NODE --ignore-daemonsets --delete-emptydir-data --force

# upgrade kubelet and kubectl
ssh $WORKER_IP
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet='1.37.*' kubectl='1.37.*'
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet

ssh $CONTROLLER
kubectl uncordon $WORKER_NODE
kubectl get nodes -o wide
```

# Tools

The CKA curriculum asks you to use Helm and Kustomize to install cluster components, for example a CNI, an ingress or Gateway controller, metrics-server, cert-manager or Argo CD.

## Kustomize

Change manifests without templates. Built into kubectl: `kubectl kustomize` shows the result, `kubectl apply -k` applies it.

### Install CRDs from a remote kustomization

```bash
# show the result first
kubectl kustomize "https://github.com/argoproj/argo-cd/manifests/crds?ref=v3.5.6" | grep '^  name:'

# these CRDs are too large for client-side apply, so use server-side apply
kubectl apply --server-side -k "https://github.com/argoproj/argo-cd/manifests/crds?ref=v3.5.6"
kubectl get crd | grep argoproj.io
```

### Install metrics-server with a patch

The upstream manifest does not have `--kubelet-insecure-tls`. Add it with a patch, without editing the downloaded file.

```bash
mkdir -p metrics-server
curl -fsSLo metrics-server/components.yaml https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.9.1/components.yaml

cat <<EOF >metrics-server/kustomization.yaml
resources:
- components.yaml
patches:
- target:
    kind: Deployment
    name: metrics-server
  patch: |-
    - op: add
      path: /spec/template/spec/containers/0/args/-
      value: --kubelet-insecure-tls
EOF

kubectl kustomize metrics-server | grep -A 7 'args:'
kubectl apply -k metrics-server
kubectl -n kube-system rollout status deployment metrics-server
kubectl top nodes
```

### Other common fields

```yaml
namespace: monitoring
namePrefix: prod-
labels:
- pairs:
    env: prod
replicas:
- name: metrics-server
  count: 2
images:
- name: registry.k8s.io/metrics-server/metrics-server
  newTag: v0.9.1
```

## Helm

Package manager for Kubernetes. A chart is a package of templates. Values change the templates.

This example installs Argo CD. The Argo CD CRDs are already installed (see Kustomize above), so the chart must not install them.

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm search repo argo/argo-cd --versions | head

# find the value to change
helm show values argo/argo-cd --version 10.11.6 > argocd-values.yaml
grep -A 2 '^crds:' argocd-values.yaml

# render templates to a file, do not install
helm template argocd argo/argo-cd --version 10.11.6 -n argocd --set crds.install=false > argocd.yaml
grep -c 'kind: CustomResourceDefinition' argocd.yaml # 0

# install with changed values
helm install argocd argo/argo-cd --version 10.11.6 -n argocd --create-namespace --set crds.install=false
kubectl -n argocd get pods

# upgrade with a values file
cat <<EOF >argocd-custom-values.yaml
crds:
  install: false
server:
  replicas: 2
EOF
helm upgrade --install argocd argo/argo-cd --version 10.11.6 -n argocd -f argocd-custom-values.yaml
kubectl -n argocd get deployment argocd-server

helm list -A
helm get values argocd -n argocd
helm history argocd -n argocd
helm rollback argocd 1 -n argocd
helm uninstall argocd -n argocd
```

## Custom Resource Definitions

A CRD adds a new resource type to the API. Operators and controllers (for example cert-manager) use CRDs.

```bash
kubectl get crd
kubectl get crd | grep cert-manager > crds.txt
kubectl get crd -o name | grep cert-manager

# show the fields of a custom resource
kubectl explain certificates.spec
kubectl explain certificates --api-version=cert-manager.io/v1

# list resources of a CRD in all namespaces
kubectl api-resources | grep cert-manager
kubectl get certificates -A
```

# Exam tips

## Example tasks

1. Create PV
2. Backup & Restore etcd
3. Troubleshoot node NotReady
4. Multi container pod with emptyDir volume
5. Create Deployment
6. Create Ingress
7. Create Pod
8. ReadinessProbe
9. Fix Service selector
10. Upgrade master node
11. Create ClusterRole + ClusterRoleBinding
12. Scale replica
13. NodeSelector
14. PVC
15. NetworkPolicy
16. Static Pod
17. Tshoot NodePort

## Example tasks (Feb 2025)

- kube-apiserver and kube-scheduler in a cluster were not working, but etcd, kube-controller-manager, and kubelet were. Troubleshoot and fix the issue.

- Create a NetworkPolicy manifest so that pods in namespace backend can only have ingress traffic from the frontend namespace.

- List all CRDs matching a keyword (cert-manager) and write them into a file.

- Expose a deployment using a NodePort service.

- Create an HPA (Horizontal Pod Autoscaler) for an existing deployment.

- Create an Ingress resource matching the scenario described in the question.

- Create a Gateway with TLS and an HTTPRoute, matching the existing Ingress resource in the environment. Delete the Ingress after creating the Gateway.

- Generate a Helm template and save it to a file using the helm template command. Then install the Helm chart with some changes to the Helm values. Both tasks were in a specific namespace, and a specific chart version was mentioned.

- Create a PriorityClass with a modification compared to an existing user-defined PriorityClass.

- Create a StorageClass.

- Create a PVC and attach it to a Pod. There was an existing PV in the environment, and we had to choose PVC properties to match the PV.

- Add a sidecar log container to an existing deployment by mounting a shared volume.

- Change the ConfigMap properties of an existing NGINX ConfigMap to enable both TLSv1.2 and TLSv1.3. Only TLSv1.3 was enabled initially.

- A deployment with three replicas had some pods in a pending state because the resource requests of containers exceeded the resources available on the node. Check the node’s CPU and memory, then divide them equally among the containers — keeping some overhead for system components and buffer — so that the deployment schedules all three replicas without any being pending.

- Choose a CNI between Flannel and Calico that has built-in support for Network Policies (Calico supports them). Install the CNI and configure it to work with the current node’s PodCIDR.
