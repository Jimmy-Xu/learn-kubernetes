run jenkins on kubernetes
========================

Test Env: MacOS 10.14.1 + minikube v0.30.0 + kubernetes v1.10.0 + jenkins 2.138.3

<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Reference](#reference)
- [Install dependency](#install-dependency)
- [Start a kubernetes cluster with minikube](#start-a-kubernetes-cluster-with-minikube)
	- [start kubernetes](#start-kubernetes)
	- [check kubernetes](#check-kubernetes)
- [Run jenkins master on kubernetes](#run-jenkins-master-on-kubernetes)
- [Use jenkins](#use-jenkins)
	- [init jenkins](#init-jenkins)
	- [install kubernetes plugin](#install-kubernetes-plugin)
	- [prepare kubernetes config in minikube vm](#prepare-kubernetes-config-in-minikube-vm)
	- [run jenkins job](#run-jenkins-job)
		- [basic job](#basic-job)
	- [pipline job](#pipline-job)
- [FAQ](#faq)
	- [external DNS does not work](#external-dns-does-not-work)

<!-- /TOC -->

# Reference
- https://www.qikqiak.com/post/kubernetes-jenkins1
- https://www.qikqiak.com/post/kubernetes-jenkins2
- https://www.qikqiak.com/post/kubernetes-jenkins3

# Install dependency

> install kubectl and minikube on MacOS

```
// install kubectl
$ brew install kubernetes-cli

// install minikube
$ curl -Lo minikube http://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/releases/v0.30.0/minikube-darwin-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/

// install hyperkit
$ curl -LO https://storage.googleapis.com/minikube/releases/latest/docker-machine-driver-hyperkit \
&& sudo install -o root -g wheel -m 4755 docker-machine-driver-hyperkit /usr/local/bin/
```

# Start a kubernetes cluster with minikube

## start kubernetes
```
// init (it will download minikube-v0.30.0.iso, kubelet and kubeadm for linux)
$ proxychains4 -q sudo minikube start --vm-driver=hyperkit --registry-mirror=https://registry.docker-cn.com

// start kubernetes via minikube
$ sudo minikube start --vm-driver=hyperkit --registry-mirror=https://registry.docker-cn.com -v10

// get kubernetes master info
$ kubectl get nodes
NAME       STATUS    ROLES     AGE       VERSION
minikube   Ready     master    34m       v1.10.0
```

## check kubernetes

check driver name
```
$ sudo cat ~/.minikube/machines/minikube/config.json | grep DriverName
    "DriverName": "hyperkit",
```

check iso
```
$ sudo cat ~/.minikube/machines/minikube/config.json | grep -i iso
        "Boot2DockerURL": "file:///Users/xjimmy/.minikube/cache/iso/minikube-v0.30.0.iso",
```

get all images
```
$ sudo minikube ssh docker images
REPOSITORY                                                                          TAG                 IMAGE ID            CREATED             SIZE
registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller        0.19.0              22ebbdddfabb        2 months ago        414MB
registry.cn-hangzhou.aliyuncs.com/google_containers/coredns                         1.2.2               367cdc8433a4        2 months ago        39.2MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64      v1.10.0             0dab2435c100        3 months ago        122MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64                v1.10.0             bfc21aadc7d3        8 months ago        97MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64   v1.10.0             ad86dbed1555        8 months ago        148MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64            v1.10.0             af20925d51a3        8 months ago        225MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64            v1.10.0             704ba848e69a        8 months ago        50.4MB
registry.cn-hangzhou.aliyuncs.com/google_containers/etcd-amd64                      3.1.12              52920ad46f5b        8 months ago        193MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-addon-manager              v8.6                9c16409588eb        9 months ago        78.4MB
registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-dns-dnsmasq-nanny-amd64     1.14.8              c2ce1ffb51ed        10 months ago       41MB
k8s.gcr.io/k8s-dns-dnsmasq-nanny-amd64                                              1.14.8              c2ce1ffb51ed        10 months ago       41MB
k8s.gcr.io/k8s-dns-sidecar-amd64                                                    1.14.8              6f7f2dc7fab5        10 months ago       42.2MB
registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-dns-sidecar-amd64           1.14.8              6f7f2dc7fab5        10 months ago       42.2MB
k8s.gcr.io/k8s-dns-kube-dns-amd64                                                   1.14.8              80cc5ea4b547        10 months ago       50.5MB
registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-dns-kube-dns-amd64          1.14.8              80cc5ea4b547        10 months ago       50.5MB
k8s.gcr.io/pause-amd64                                                              3.1                 da86e6ba6ca1        11 months ago       742kB
k8s.gcr.io/pause                                                                    3.1                 da86e6ba6ca1        11 months ago       742kB
registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64                     3.1                 da86e6ba6ca1        11 months ago       742kB
registry.cn-hangzhou.aliyuncs.com/google_containers/pause                           3.1                 da86e6ba6ca1        11 months ago       742kB
registry.cn-hangzhou.aliyuncs.com/google_containers/storage-provisioner             v1.8.1              4689081edb10        12 months ago       80.8MB
registry.cn-hangzhou.aliyuncs.com/google_containers/defaultbackend                  1.4                 846921f0fe0e        13 months ago       4.84MB
registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-dns-sidecar-amd64           1.14.5              fed89e8b4248        14 months ago       41.8MB
registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-dns-kube-dns-amd64          1.14.5              512cd7425a73        14 months ago       49.4MB
registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-dns-dnsmasq-nanny-amd64     1.14.5              459944ce8cc4        14 months ago       41.4MB
```

get all resources (pod should be Running)
```
$ kubectl -n kube-system get all
NAME                                            READY     STATUS    RESTARTS   AGE
pod/coredns-5bfd87b64b-9gv9g                    1/1       Running   0          41m
pod/default-http-backend-5594468747-b89n7       1/1       Running   0          41m
pod/etcd-minikube                               1/1       Running   0          41m
pod/kube-addon-manager-minikube                 1/1       Running   0          41m
pod/kube-apiserver-minikube                     1/1       Running   0          40m
pod/kube-controller-manager-minikube            1/1       Running   0          41m
pod/kube-dns-b4bd9576-rw5w7                     3/3       Running   0          41m
pod/kube-proxy-z77hg                            1/1       Running   0          41m
pod/kube-scheduler-minikube                     1/1       Running   0          41m
pod/kubernetes-dashboard-866c7586d-qs7m8        1/1       Running   0          41m
pod/nginx-ingress-controller-7445c5d647-fgxrh   1/1       Running   0          41m
pod/storage-provisioner                         1/1       Running   0          41m

NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
service/default-http-backend   NodePort    10.99.183.122   <none>        80:30001/TCP    41m
service/kube-dns               ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP   41m
service/kubernetes-dashboard   ClusterIP   10.98.109.25    <none>        80/TCP          41m

NAME                        DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/kube-proxy   1         1         1         1            1           <none>          41m

NAME                                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coredns                    1         1         1            1           41m
deployment.apps/default-http-backend       1         1         1            1           41m
deployment.apps/kube-dns                   1         1         1            1           41m
deployment.apps/kubernetes-dashboard       1         1         1            1           41m
deployment.apps/nginx-ingress-controller   1         1         1            1           41m

NAME                                                  DESIRED   CURRENT   READY     AGE
replicaset.apps/coredns-5bfd87b64b                    1         1         1         41m
replicaset.apps/default-http-backend-5594468747       1         1         1         41m
replicaset.apps/kube-dns-b4bd9576                     1         1         1         41m
replicaset.apps/kubernetes-dashboard-866c7586d        1         1         1         41m
replicaset.apps/nginx-ingress-controller-7445c5d647   1         1         1         41m
```

# Run jenkins master on kubernetes
```
$ kubectl create namespace kube-ops

$ kubectl create -f rbac.yaml

$ kubectl create -f jenkins2.yaml

$ kubectl -n kube-ops get all
NAME                            READY     STATUS    RESTARTS   AGE
pod/jenkins2-64874dddb7-c4cg5   1/1       Running   0          9h

NAME               TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)                          AGE
service/jenkins2   NodePort   10.100.3.71   <none>        8080:30002/TCP,50000:32347/TCP   9h

NAME                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/jenkins2   1         1         1            1           9h

NAME                                  DESIRED   CURRENT   READY     AGE
replicaset.apps/jenkins2-64874dddb7   1         1         1         9h
```

# Use jenkins

##  init jenkins

> Access jenkins on host

get jenkins init password
```
$ POD=$(kubectl -n kube-ops get pods  -o name)

$ kubectl -n kube-ops logs $POD | grep installation -A2
Please use the following password to proceed to installation:

c01c618eaaae487aa2ee27f6a07e1bd3
```

get service for jenkins
```
$ kubectl -n kube-ops get services
NAME       TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)                          AGE
jenkins2   NodePort   10.100.3.71   <none>        8080:30002/TCP,50000:32347/TCP   34m
```

open jenkins web ui from MacOS
```
// get vm ip
$ sudo minikube ip
192.168.64.37

// open https://$(sudo minikube ip):30002 in web browser
```

 init jenkins in web ui
```
- Unlock jenkins with then above init password
- Install suggested plugins
- Create first admin user
- set Jenkins URL:  http://192.168.64.37:30002  (use default)
```

## install kubernetes plugin

get kubernetes config
```
$ kubectl config view | grep ".minikube/ca.crt" -A1
    certificate-authority: /Users/xjimmy/.minikube/ca.crt
    server: https://192.168.64.37:8443
```

install kubernetes plugins in kenkins
```
- goto plugin manager: http://192.168.64.37:30002/pluginManager/
- filter by "kubernetes" in "Available" List
- select "Kubernetes" plugin, download plugin, select restart jenkins after install completion
```

config kubernetes plugin
```
go to Manage Jenkins - Config System:

//click "Add a new cloud", select "Kubernetes"

//config kubernetes
- set Kubernetes URL:  https://192.168.64.37:8443
- Kubernetes server certificate key: <content of /Users/xjimmy/.minikube/ca.crt>
- Kubernetes Namespace: kube-ops
- Jenkins URL: http://10.100.3.71:8080  (important: use the ip of jenkins2 service)

//Click Test Connection
- Connection test successful
```

add pod template
```
// click "Add Pod Template"

Images:
  - Kubernetes Pod template  
    - Name: jnlp
    - Namespace: kube-ops
    - Labels: jenkins-k8s
    - Usage: "Only build jobs with label expressions matching this node"
    - Container Template
      - Name: jnlp
      - Docker image: cnych/jenkins:jnlp (with kubectl)
      - Command to run: <empty>
      - Arguments to pass to the command: <empty>
    - Volumes
      - Host Path Volume:
        - Host Path: /var/run/docker.sock
        - Mount Path: /var/run/docker.sock
      - Host Path Volume:
        - HostPath: /home/docker/.kube  (in minikube vm)
        - Mount Path: /home/jenkins/.kube (in container)
```

## prepare kubernetes config in minikube vm

copy cert file from MacOS to minikube vm
```
$ sudo minikube mkdir ~/.kube
$ sudo scp -i $(sudo minikube ssh-key) -r ~/.minikube/{ca.crt,client.crt,client.key} docker@$(sudo minikube ip):/home/docker/.kube/
ca.crt                                   100% 1066     2.1MB/s   00:00
client.crt                               100% 1103     2.1MB/s   00:00
client.key                               100% 1675     3.3MB/s   00:00
```

enter minikube vm
```
$ sudo minikube ssh
```

create /home/docker/.kube/config (from host ~/.kube/config)
```
$ cat > /home/docker/.kube/config <<EOF
apiVersion: v1
clusters:
- cluster:
    certificate-authority: ca.crt
    server: https://192.168.64.37:8443
  name: minikube
- context:
    cluster: minikube
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    client-certificate: client.crt
    client-key: client.key
EOF
```

ensure file mode (owner should be 'docker')
```
$ ls -l ~/.kube
total 16
-rw-r--r-- 1 docker docker 1066 Nov 24 14:13 ca.crt
-rw-r--r-- 1 docker docker 1103 Nov 24 14:13 client.crt
-rw------- 1 docker docker 1675 Nov 24 14:13 client.key
-rw-r--r-- 1 docker docker  356 Nov 24 23:38 config
```

## run jenkins job

### basic job

> In jenkins web ui

Create and config jobs - General
```
Restrict where this project can be run:  jenkins-k8s
```

Add build step - Execute shell :
```
echo "测试 Kubernetes 动态生成 jenkins slave"
echo "==============docker in docker==========="
docker info

echo "=============kubectl============="
kubectl get pods
```

Console output
```
Started by user admin
Agent jnlp-rfp1b is provisioned from template Kubernetes Pod Template
Agent specification [Kubernetes Pod Template] (jenkins-k8s):
* [jnlp] cnych/jenkins:jnlp

Building remotely on jnlp-rfp1b (jenkins-k8s) in workspace /home/jenkins/workspace/test
[test] $ /bin/sh -xe /tmp/jenkins6736958880969564739.sh
+ echo 测试 Kubernetes 动态生成 jenkins slave
测试 Kubernetes 动态生成 jenkins slave
+ echo ==============docker in docker===========
==============docker in docker===========
+ docker info
Containers: 30
 Running: 30
 Paused: 0
 Stopped: 0
Images: 17
Server Version: 17.12.1-ce
Storage Driver: overlay2
 Backing Filesystem: extfs
 Supports d_type: true
 Native Overlay Diff: true
Logging Driver: json-file
Cgroup Driver: cgroupfs
Plugins:
 Volume: local
 Network: bridge host macvlan null overlay
 Log: awslogs fluentd gcplogs gelf journald json-file logentries splunk syslog
Swarm: inactive
Runtimes: runc
Default Runtime: runc
Init Binary: docker-init
containerd version: 9b55aab90508bd389d7654c4baf173a981477d55
runc version: 9f9c96235cc97674e935002fc3d78361b696a69e
init version: N/A (expected: )
Security Options:
 seccomp
  Profile: default
Kernel Version: 4.15.0
Operating System: Buildroot 2018.05
OSType: linux
Architecture: x86_64
CPUs: 2
Total Memory: 1.944GiB
Name: minikube
ID: YGSY:WF3V:5EXZ:YUHV:RFYW:FZ2B:WWTT:UTED:SQYR:GQW3:ER2H:SH4K
Docker Root Dir: /var/lib/docker
Debug Mode (client): false
Debug Mode (server): false
Registry: https://index.docker.io/v1/
Labels:
 provider=hyperkit
Experimental: false
Insecure Registries:
 10.96.0.0/12
 127.0.0.0/8
Registry Mirrors:
 https://registry.docker-cn.com/
Live Restore Enabled: false

+ echo =============kubectl=============
=============kubectl=============
+ kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
jenkins2-64874dddb7-9w9zk   1/1       Running   0          8h
jnlp-rfp1b                  1/1       Running   0          11s
Finished: SUCCESS
```

## pipline job

```

```


# FAQ

## external DNS does not work

error
```
"Could not resolve host: github.com" when run job in container
```

solution
```
// enable dns
$ sudo minikube addons enable kube-dns
$ sudo minikube addons list

$ kubectl -n kube-system get svc kube-dns
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP   19d

$ sudo minikube ssh
$ sudo vi /etc/resolv.conf
// add a line in resolv.conf
nameserver 10.96.0.10

$ nslookup kube-dns.kube-system.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kube-dns.kube-system.svc.cluster.local
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local


// add in job before clone repo from github
$ bash -c "echo nameserver 8.8.8.8 | sudo tee -a /etc/resolv.conf"
```
