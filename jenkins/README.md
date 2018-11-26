run jenkins on kubernetes
========================

Test Env: MacOS 10.14.1 + minikube v0.30.0 + kubernetes v1.10.0 + jenkins 2.138.3

<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Reference](#reference)
- [Install dependency](#install-dependency)
- [Start a single node kubernetes with minikube](#start-a-single-node-kubernetes-with-minikube)
	- [start kubernetes](#start-kubernetes)
	- [check kubernetes](#check-kubernetes)
	- [backup image for kubernetes in minikube](#backup-image-for-kubernetes-in-minikube)
- [Run jenkins master on kubernetes](#run-jenkins-master-on-kubernetes)
- [Use jenkins](#use-jenkins)
	- [init jenkins](#init-jenkins)
	- [install kubernetes plugin](#install-kubernetes-plugin)
	- [prepare kubernetes config in minikube vm](#prepare-kubernetes-config-in-minikube-vm)
	- [run jenkins job](#run-jenkins-job)
		- [basic job](#basic-job)
		- [pipline job](#pipline-job)

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

# Start a single node kubernetes with minikube

## start kubernetes
```
// start minikube first  (minikube-v0.30.0.iso, kubelet and kubeadm will be download via proxychains4)
$ proxychains4 -q sudo minikube start --vm-driver=hyperkit --registry-mirror=https://registry.docker-cn.com -v10
$ sudo minikube delete

// start minikube second  (image will be pulled via registry-mirror, change /var/lib/docker to /mnt/vda1/docker)
// - push image via http proxy
// - pull image via registry-mirror
$ sudo minikube ssh "sudo mkdir -p /mnt/vda1/docker"
$ sudo minikube start --vm-driver=hyperkit --registry-mirror=https://registry.docker-cn.com -v=10 \
  --disk-size=20g --docker-opt="data-root=/mnt/vda1/docker" \
  --docker-env=HTTP_PROXY=http://192.168.64.1:8118 \
  --docker-env=HTTPS_PROXY=http://192.168.64.1:8118 \
  --docker-env=NO_PROXY=/var/run/docker.sock,127.0.0.1,registry.docker-cn.com,registry.cn-hangzhou.aliyuncs.com,registry.cn-beijing.aliyuncs.com,registry.cn-shanghai.aliyuncs.com,dockerauth.cn-hangzhou.aliyuncs.com,dockerauth.cn-beijing.aliyuncs.com,dockerauth.cn-shanghai.aliyuncs.com,aliregistry.oss-cn-hangzhou.aliyuncs.com,aliregistry.oss-cn-beijing.aliyuncs.com,aliregistry.oss-cn-shanghai.aliyuncs.com,registry-mirror-cache-cn.oss-cn-hangzhou.aliyuncs.com,registry-mirror-cache-cn.oss-cn-beijing.aliyuncs.com,registry-mirror-cache-cn.oss-cn-shanghai.aliyuncs.com

// get kubernetes master info
$ kubectl get nodes
NAME       STATUS    ROLES     AGE       VERSION
minikube   Ready     master    34m       v1.10.0

// check pods (should be Running)
$ kubectl get pods -n kube-system
NAME                                        READY     STATUS    RESTARTS   AGE
coredns-5bfd87b64b-9lkrl                    1/1       Running   0          28m
default-http-backend-5594468747-8scrz       1/1       Running   0          28m
etcd-minikube                               1/1       Running   0          27m
kube-addon-manager-minikube                 1/1       Running   0          28m
kube-apiserver-minikube                     1/1       Running   1          27m
kube-controller-manager-minikube            1/1       Running   0          28m
kube-dns-b4bd9576-bt227                     3/3       Running   1          28m
kube-proxy-g5z7f                            1/1       Running   0          28m
kube-scheduler-minikube                     1/1       Running   0          28m
kubernetes-dashboard-866c7586d-7njwg        1/1       Running   0          28m
nginx-ingress-controller-7445c5d647-t4bf5   1/1       Running   0          28m
storage-provisioner                         1/1       Running   0          28m
```

## check kubernetes

check driver name
```
$ sudo cat ~/.minikube/machines/minikube/config.json | grep DriverName
    "DriverName": "hyperkit",
```

check registry-mirror
```
$ sudo grep -A2 RegistryMirror ~/.minikube/machines/minikube/config.json
            "RegistryMirror": [
                "https://registry.docker-cn.com"
            ],
```

check http proxy for docker
```
$ sudo grep -C1 PROXY ~/.minikube/machines/minikube/config.json
            "Env": [
                "HTTP_PROXY=http://192.168.64.1:8118",
                "HTTPS_PROXY=http://192.168.64.1:8118",
                "NO_PROXY=/var/run/docker.sock,127.0.0.1,registry.docker-cn.com,registry.cn-hangzhou.aliyuncs.com,registry.cn-beijing.aliyuncs.com,registry.cn-shanghai.aliyuncs.com,dockerauth.cn-hangzhou.aliyuncs.com,dockerauth.cn-beijing.aliyuncs.com,dockerauth.cn-shanghai.aliyuncs.com,aliregistry.oss-cn-hangzhou.aliyuncs.com,aliregistry.oss-cn-beijing.aliyuncs.com,aliregistry.oss-cn-shanghai.aliyuncs.com,registry-mirror-cache-cn.oss-cn-hangzhou.aliyuncs.com,registry-mirror-cache-cn.oss-cn-beijing.aliyuncs.com,registry-mirror-cache-cn.oss-cn-shanghai.aliyuncs.com"
            ],
```

check iso
```
$ sudo cat ~/.minikube/machines/minikube/config.json | grep -i iso
        "Boot2DockerURL": "file:///Users/xjimmy/.minikube/cache/iso/minikube-v0.30.0.iso",
```

check docker in minikube vm
```
$ sudo minikube ssh docker info
```

check fs
```
$ sudo minikube ssh "sudo df -hT"
Filesystem     Type      Size  Used Avail Use% Mounted on
rootfs         rootfs    913M  749M  165M  82% /
devtmpfs       devtmpfs  913M     0  913M   0% /dev
tmpfs          tmpfs     996M     0  996M   0% /dev/shm
tmpfs          tmpfs     996M  9.1M  987M   1% /run
tmpfs          tmpfs     996M     0  996M   0% /sys/fs/cgroup
tmpfs          tmpfs     996M  8.0K  996M   1% /tmp
/dev/vda1      ext4       16G  2.7G   12G  19% /mnt/vda1
overlay        overlay    16G  2.7G   12G  19% /mnt/vda1/docker/overlay2/08859c04135df597ed8b172a6b1963aa84c7c68d3437865ad03790cd75f43f27/merged
tmpfs          tmpfs     996M   12K  996M   1% /var/lib/kubelet/pods/053a5ccd-f20f-11e8-839e-e687a56af0a1/volumes/kubernetes.io~secret/kube-proxy-token-42zpj
...
```

get all docker images
```
$ sudo minikube ssh docker images
REPOSITORY                                                                          TAG                 IMAGE ID            CREATED             SIZE
registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller        0.19.0              22ebbdddfabb        2 months ago        414MB
registry.cn-hangzhou.aliyuncs.com/google_containers/coredns                         1.2.2               367cdc8433a4        2 months ago        39.2MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64      v1.10.0             0dab2435c100        3 months ago        122MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64                v1.10.0             bfc21aadc7d3        8 months ago        97MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64            v1.10.0             af20925d51a3        8 months ago        225MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64   v1.10.0             ad86dbed1555        8 months ago        148MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64            v1.10.0             704ba848e69a        8 months ago        50.4MB
registry.cn-hangzhou.aliyuncs.com/google_containers/etcd-amd64                      3.1.12              52920ad46f5b        8 months ago        193MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-addon-manager              v8.6                9c16409588eb        9 months ago        78.4MB
k8s.gcr.io/k8s-dns-dnsmasq-nanny-amd64                                              1.14.8              c2ce1ffb51ed        10 months ago       41MB
registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-dns-dnsmasq-nanny-amd64     1.14.8              c2ce1ffb51ed        10 months ago       41MB
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
```

get all k8s resources (pod should be Running)
```
$ kubectl -n kube-system get all
NAME                                            READY     STATUS    RESTARTS   AGE
pod/coredns-5bfd87b64b-9lkrl                    1/1       Running   0          30m
pod/default-http-backend-5594468747-8scrz       1/1       Running   0          30m
pod/etcd-minikube                               1/1       Running   0          29m
pod/kube-addon-manager-minikube                 1/1       Running   0          30m
pod/kube-apiserver-minikube                     1/1       Running   1          29m
pod/kube-controller-manager-minikube            1/1       Running   0          30m
pod/kube-dns-b4bd9576-bt227                     3/3       Running   1          30m
pod/kube-proxy-g5z7f                            1/1       Running   0          30m
pod/kube-scheduler-minikube                     1/1       Running   0          30m
pod/kubernetes-dashboard-866c7586d-7njwg        1/1       Running   0          30m
pod/nginx-ingress-controller-7445c5d647-t4bf5   1/1       Running   0          30m
pod/storage-provisioner                         1/1       Running   0          30m

NAME                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
service/default-http-backend   NodePort    10.110.27.184    <none>        80:30001/TCP    30m
service/kube-dns               ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP   30m
service/kubernetes-dashboard   ClusterIP   10.108.119.187   <none>        80/TCP          30m

NAME                        DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/kube-proxy   1         1         1         1            1           <none>          30m

NAME                                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coredns                    1         1         1            1           30m
deployment.apps/default-http-backend       1         1         1            1           30m
deployment.apps/kube-dns                   1         1         1            1           30m
deployment.apps/kubernetes-dashboard       1         1         1            1           30m
deployment.apps/nginx-ingress-controller   1         1         1            1           30m

NAME                                                  DESIRED   CURRENT   READY     AGE
replicaset.apps/coredns-5bfd87b64b                    1         1         1         30m
replicaset.apps/default-http-backend-5594468747       1         1         1         30m
replicaset.apps/kube-dns-b4bd9576                     1         1         1         30m
replicaset.apps/kubernetes-dashboard-866c7586d        1         1         1         30m
replicaset.apps/nginx-ingress-controller-7445c5d647   1         1         1         30m
```

check shadowsocks log in MacOS
> check the url when pull image
```
$ tail -f ~/Library/Logs/ss-local.log
```

## backup image for kubernetes in minikube

to speed up pull image

- backup images
- copy image to host
- load images when start minikube later

```
//Image should been pulled successfully at least once

// create dir
$ sudo minikube ssh "sudo mkdir -p /mnt/vda1/image; sudo chown docker:docker /mnt/vda1/image"

//enter minikube vm
$ sudo minikube ssh
$ sudo -s
$ cd /mnt/vda1/image
$ mkdir -p registry.cn-hangzhou.aliyuncs.com/google_containers k8s.gcr.io
// generate docker save command line, then run manually
$ docker images | grep -v REPOSITORY | awk '{printf "docker save %s:%s | gzip> %s_%s.tar.gz\n",$1,$2,$1,$2}'
docker save registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:0.19.0 | gzip> registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller_0.19.0.tar.gz
...
docker save k8s.gcr.io/k8s-dns-kube-dns-amd64:1.14.8 | gzip> k8s.gcr.io/k8s-dns-kube-dns-amd64_1.14.8.tar.gz
$ tar cvf minikube-docker-image.tar k8s.gcr.io/ registry.cn-hangzhou.aliyuncs.com/

//exit minikube vm, backup image backup file from minikube vm to host
$ sudo scp -i $(sudo minikube ssh-key) docker@$(sudo minikube ip):/mnt/vda1/image/minikube-docker-image.tar ~/minikube/

//to copy image from host to vm
$ sudo scp -i $(sudo minikube ssh-key) ~/minikube/minikube-docker-image.tar docker@$(sudo minikube ip):/mnt/vda1/image

//to load image
$ find . -type f | xargs -i docker load -i {}
```

# Run jenkins master on kubernetes

use host path as volume of jenkins
```
$ sudo minikube ssh 'bash -c "sudo mkdir -p /mnt/vda1/data/jenkins; sudo chown 1000:1000 /mnt/vda1/data/jenkins"'
```

pull jenkins Images
```
$ sudo minikube ssh
$ sudo -s
$ docker pull jenkins/jenkins:lts
$ docker pull cnych/jenkins:jnlp
$ cd /mnt/vda1/image/
$ docker save jenkins/jenkins:lts cnych/jenkins:jnlp | gzip > jenkins-docker-image.tar.gz

//exit minikube vm, copy file from vm to host
$ sudo scp -i $(sudo minikube ssh-key) docker@$(sudo minikube ip):/mnt/vda1/image/jenkins-docker-image.tar.gz ~/minikube/
```

run jenkins from yaml
```
$ kubectl create namespace kube-ops

$ cd $GOPATH/src/github.com/jimmy-xu
$ git clone https://github.com/Jimmy-Xu/learn-kubernetes.git
$ cd $GOPATH/src/github.com/jimmy-xu/learn-kubernetes/jenkins

$ kubectl create -f rbac.yaml

$ kubectl create -f jenkins2.yaml

$ kubectl -n kube-ops get all
NAME                            READY     STATUS    RESTARTS   AGE
pod/jenkins2-6c757dd7df-lmmg7   0/1       Running   0          5s

NAME               TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)                          AGE
service/jenkins2   NodePort   10.96.14.75   <none>        8080:30002/TCP,50000:31441/TCP   5s

NAME                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/jenkins2   1         1         1            0           5s

NAME                                  DESIRED   CURRENT   READY     AGE
replicaset.apps/jenkins2-6c757dd7df   1         1         0         5s
```

check volume data
```
$ sudo minikube ssh 'bash -c "ls -l /mnt/vda1/data/jenkins"'
total 76
-rw-r--r--  1 rkt rkt 1643 Nov 27 07:25 config.xml
-rw-r--r--  1 rkt rkt  102 Nov 27 07:24 copy_reference_file.log
-rw-r--r--  1 rkt rkt  156 Nov 27 07:25 hudson.model.UpdateCenter.xml
-rw-------  1 rkt rkt 1712 Nov 27 07:25 identity.key.enc
drwxr-xr-x  2 rkt rkt 4096 Nov 27 07:24 init.groovy.d
-rw-r--r--  1 rkt rkt   94 Nov 27 07:25 jenkins.CLI.xml
-rw-r--r--  1 rkt rkt    7 Nov 27 07:25 jenkins.install.UpgradeWizard.state
-rw-r--r--  1 rkt rkt  171 Nov 27 07:25 jenkins.telemetry.Correlator.xml
drwxr-xr-x  2 rkt rkt 4096 Nov 27 07:25 jobs
drwxr-xr-x  3 rkt rkt 4096 Nov 27 07:25 logs
-rw-r--r--  1 rkt rkt  907 Nov 27 07:25 nodeMonitors.xml
drwxr-xr-x  2 rkt rkt 4096 Nov 27 07:25 nodes
drwxr-xr-x  2 rkt rkt 4096 Nov 27 07:25 plugins
-rw-r--r--  1 rkt rkt   64 Nov 27 07:25 secret.key
-rw-r--r--  1 rkt rkt    0 Nov 27 07:25 secret.key.not-so-secret
drwx------  4 rkt rkt 4096 Nov 27 07:25 secrets
drwxr-xr-x  2 rkt rkt 4096 Nov 27 07:25 updates
drwxr-xr-x  2 rkt rkt 4096 Nov 27 07:25 userContent
drwxr-xr-x  3 rkt rkt 4096 Nov 27 07:25 users
drwxr-xr-x 11 rkt rkt 4096 Nov 27 07:24 war
```

# Use jenkins

##  init jenkins

> Access jenkins on MacOS

get jenkins init password
```
$ POD=$(kubectl -n kube-ops get pods  -o name)

$ kubectl -n kube-ops logs $POD | grep installation -A2
Please use the following password to proceed to installation:

37e9edb77b0e4d91a0e9731b7ef18973
```

get service for jenkins
```
$ kubectl -n kube-ops get services
NAME       TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)                          AGE
jenkins2   NodePort   10.96.14.75   <none>        8080:30002/TCP,50000:31441/TCP   1m
```

open jenkins web ui from MacOS
```
// get vm ip
$ sudo minikube ip
192.168.64.50

// open https://$(sudo minikube ip):30002 in web browser
```

 init jenkins in web ui
```
- Unlock jenkins with then above init password
- Install suggested plugins
- Create first admin user
- set Jenkins URL:  http://192.168.64.50:30002  (use default)
```

## install kubernetes plugin

get kubernetes config
```
$ kubectl config view | grep ".minikube/ca.crt" -A1
    certificate-authority: /Users/xjimmy/.minikube/ca.crt
    server: https://192.168.64.50:8443
```

install kubernetes plugins in kenkins
```
- goto plugin manager: http://192.168.64.50:30002/pluginManager/
- filter by "kubernetes" in "Available" List
- select "Kubernetes" plugin, download plugin, select restart jenkins after install completion
```

config kubernetes plugin
```
go to Manage Jenkins - Config System:

//click "Add a new cloud", select "Kubernetes"

//config kubernetes
- set Kubernetes URL:  https://192.168.64.50:8443
- Kubernetes server certificate key: <content of /Users/xjimmy/.minikube/ca.crt>
- Kubernetes Namespace: kube-ops
//Click Test Connection, the result should be "Connection test successful"
- Jenkins URL: http://10.96.14.75:8080  (important: use the ip of jenkins2 service)
```

add pod template
```
// click "Add Pod Template"

- Kubernetes Pod template  
  - Name: jnlp
  - Namespace: kube-ops
  - Labels: jenkins-k8s
  - Usage: "Only build jobs with label expressions matching this node"
  - Container Template
    - Name: jnlp
    - Docker image: cnych/jenkins:jnlp (base on debian, with kubectl and docker installed)
    - Command to run: <empty>
    - Arguments to pass to the command: <empty>
  - Volumes
    - Host Path Volume:
      - Host Path: /var/run/docker.sock (use docker on minikube vm)
      - Mount Path: /var/run/docker.sock
    - Host Path Volume:
      - HostPath: /home/docker/.kube  (on minikube vm)
      - Mount Path: /home/jenkins/.kube (in container)
  - Time in seconds for Pod deadline: 100
```

## prepare kubernetes config in minikube vm

copy cert file from MacOS to minikube vm
```
$ sudo minikube ssh "mkdir ~/.kube"
$ sudo scp -i $(sudo minikube ssh-key) -r ~/.minikube/{ca.crt,client.crt,client.key} docker@$(sudo minikube ip):/home/docker/.kube/
ca.crt                                   100% 1066     2.1MB/s   00:00
client.crt                               100% 1103     2.1MB/s   00:00
client.key                               100% 1675     3.3MB/s   00:00
```

create /home/docker/.kube/config (from host ~/.kube/config)
```
$ sudo minikube ssh

$ cat > /home/docker/.kube/config <<EOF
apiVersion: v1
clusters:
- cluster:
    certificate-authority: ca.crt
    server: https://192.168.64.50:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    user: minikube
  name: minikube
current-context: minikube
kind: Config
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
ID: G55M:A3A5:NZMT:M3HA:Z3Q5:NEYX:KW6S:2KYN:22RD:5MFC:KJQK:M27O
Docker Root Dir: /mnt/vda1/docker
Debug Mode (client): false
Debug Mode (server): false
Http Proxy: http://192.168.64.1:8118
Https Proxy: http://192.168.64.1:8118
No Proxy: /var/run/docker.sock,127.0.0.1,registry.docker-cn.com,registry.cn-hangzhou.aliyuncs.com,registry.cn-beijing.aliyuncs.com,registry.cn-shanghai.aliyuncs.com,dockerauth.cn-hangzhou.aliyuncs.com,dockerauth.cn-beijing.aliyuncs.com,dockerauth.cn-shanghai.aliyuncs.com,aliregistry.oss-cn-hangzhou.aliyuncs.com,aliregistry.oss-cn-beijing.aliyuncs.com,aliregistry.oss-cn-shanghai.aliyuncs.com,registry-mirror-cache-cn.oss-cn-hangzhou.aliyuncs.com,registry-mirror-cache-cn.oss-cn-beijing.aliyuncs.com,registry-mirror-cache-cn.oss-cn-shanghai.aliyuncs.com
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
jenkins2-6c757dd7df-lmmg7   1/1       Running   0          43m
jnlp-vt2rf                  1/1       Running   0          7s
Finished: SUCCESS
```

### pipline job

create a pipeline job

add global credentials for dockerhub
```
// open global credentials, click "Add Credentials"
http://192.168.64.50:30002/credentials/store/system/domain/_/

Kind: Username with password
Username: <dockerhub username>
Password: <dockerhub password>
ID: dockerHub
Description: dockerhub user auth
```

add pipline script: https://github.com/Jimmy-Xu/jenkins-demo/blob/master/Jenkinsfile
- add github.com to /etc/hosts
- use http_proxy to clone github repo
