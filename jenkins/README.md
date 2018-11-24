run jenkins on kubernetes
========================

Test Env: MacOS 10.14.1 + minikube v0.30.0 + kubernetes v1.10.0 + jenkins

# reference
- https://www.qikqiak.com/post/kubernetes-jenkins1
- https://www.qikqiak.com/post/kubernetes-jenkins2
- https://www.qikqiak.com/post/kubernetes-jenkins3

# install dependency

> install kubectl and minikube in MacOS

```
// install kubectl
$ brew install kubernetes-cli

// install minikube
$ curl -Lo minikube http://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/releases/v0.30.0/minikube-darwin-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```

# start minikube
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

# check kubernetes

## get all images
```
$ sudo minikube ssh docker images
REPOSITORY                                                                          TAG                 IMAGE ID            CREATED             SIZE
jenkins/jenkins                                                                     lts                 d7c5abfe8477        2 weeks ago         703MB
registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller        0.19.0              22ebbdddfabb        2 months ago        414MB
registry.cn-hangzhou.aliyuncs.com/google_containers/coredns                         1.2.2               367cdc8433a4        2 months ago        39.2MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64      v1.10.0             0dab2435c100        3 months ago        122MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64                v1.10.0             bfc21aadc7d3        8 months ago        97MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64            v1.10.0             704ba848e69a        8 months ago        50.4MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64   v1.10.0             ad86dbed1555        8 months ago        148MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64            v1.10.0             af20925d51a3        8 months ago        225MB
registry.cn-hangzhou.aliyuncs.com/google_containers/etcd-amd64                      3.1.12              52920ad46f5b        8 months ago        193MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-addon-manager              v8.6                9c16409588eb        9 months ago        78.4MB
registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-dns-dnsmasq-nanny-amd64     1.14.8              c2ce1ffb51ed        10 months ago       41MB
k8s.gcr.io/k8s-dns-dnsmasq-nanny-amd64                                              1.14.8              c2ce1ffb51ed        10 months ago       41MB
k8s.gcr.io/k8s-dns-sidecar-amd64                                                    1.14.8              6f7f2dc7fab5        10 months ago       42.2MB
registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-dns-sidecar-amd64           1.14.8              6f7f2dc7fab5        10 months ago       42.2MB
k8s.gcr.io/k8s-dns-kube-dns-amd64                                                   1.14.8              80cc5ea4b547        10 months ago       50.5MB
registry.cn-hangzhou.aliyuncs.com/google_containers/k8s-dns-kube-dns-amd64          1.14.8              80cc5ea4b547        10 months ago       50.5MB
registry.cn-hangzhou.aliyuncs.com/google_containers/pause                           3.1                 da86e6ba6ca1        11 months ago       742kB
k8s.gcr.io/pause-amd64                                                              3.1                 da86e6ba6ca1        11 months ago       742kB
k8s.gcr.io/pause                                                                    3.1                 da86e6ba6ca1        11 months ago       742kB
registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64                     3.1                 da86e6ba6ca1        11 months ago       742kB
registry.cn-hangzhou.aliyuncs.com/google_containers/storage-provisioner             v1.8.1              4689081edb10        12 months ago       80.8MB
registry.cn-hangzhou.aliyuncs.com/google_containers/defaultbackend                  1.4                 846921f0fe0e        13 months ago       4.84MB
```

## get all resources
```
$ kubectl get all -n kube-system
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

# run jenkins in kubernetes
```
$ kubectl create namespace kube-ops

$ kubectl create -f rbac.yaml

$ kubectl create -f jenkins2.yaml

$ kubectl get all -n kube-ops
NAME                            READY     STATUS    RESTARTS   AGE
pod/jenkins2-64874dddb7-9w9zk   1/1       Running   0          16m

NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                          AGE
service/jenkins2   NodePort   10.103.230.57   <none>        8080:30002/TCP,50000:30995/TCP   16m

NAME                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/jenkins2   1         1         1            1           16m

NAME                                  DESIRED   CURRENT   READY     AGE
replicaset.apps/jenkins2-64874dddb7   1         1         1         16m
```

# access jenkins on host

## get jenkins init password
```
$ kubectl get pods -n kube-ops
NAME                        READY     STATUS    RESTARTS   AGE
jenkins2-64874dddb7-9w9zk   1/1       Running   0          43m

$ kubectl logs jenkins2-64874dddb7-9w9zk -n kube-ops | grep installation -A2
Please use the following password to proceed to installation:

9ee684dfb77e4b9ca917847bcc02c907
```

## get service for jenkins
```
$ kubectl get service -n kube-ops
NAME       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                          AGE
jenkins2   NodePort   10.103.230.57   <none>        8080:30002/TCP,50000:30995/TCP   45m
```

## open jenkins web ui
```
$ sudo minikube ip
192.168.64.36

$ curl -L http://`sudo minikube ip`:30002
<html><head><meta http-equiv='refresh' content='1;url=/login?from=%2F'/><script>window.location.replace('/login?from=%2F');</script></head><body style='background-color:white; color:white;'>


Authentication required
<!--
You are authenticated as: anonymous
Groups that you are in:

Permission you need to have (but didn't): hudson.model.Hudson.Administer
-->

</body></html>
```
