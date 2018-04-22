# Kubeadm安装集群

官方文档：[https://kubernetes.io/docs/setup/independent/install-kubeadm/](https://kubernetes.io/docs/setup/independent/install-kubeadm/)

## 安装Docker 1.12

```bash
sudo -s
curl https://releases.rancher.com/install-docker/1.12.sh | sh
```

脚本来自Rancher [https://rancher.com/docs/rancher/v1.6/en/hosts/\#supported-docker-versions](https://rancher.com/docs/rancher/v1.6/en/hosts/#supported-docker-versions)

## 安装kubeadm和kubectl

这里需要翻墙，参考 [Ubuntu](https://doc.tanmer.cn/ubuntu) 中的翻墙技巧

```bash
apt-get update && apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
```

检查`Docker`是否使用`Cgroup Driver`

```bash
docker info | grep -i cgroup
cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

为了让docker和k8s的Cgroup Driver保持一致，在参数后面加上 `--cgroup-driver=cgroupfs`

## 创建Master节点

创建之前，先关闭swap

```bash
sudo swapoff -a
# 注释掉swap分区
sudo vi /etc/fstab
```

### 预初始化集群

{% hint style="info" %}
这里说预初始化集群，是因为这次不是真正的初始化，只是借助他找出依赖的Docker镜像
{% endhint %}

因为k8s的镜像都在Google服务器行，我们不翻墙是无法访问的。

```bash
sudo kubeadm init
```

### 监视Docker日志

执行之后，新开窗口，监控docker日志，记录哪些镜像下载失败，并记录下来

```bash
journalctl -u docker -f
```

输出大概是这样的

```text
Apr 21 21:40:25 node1 dockerd[1144]: time="2018-04-21T21:40:25.098887386+08:00" level=error msg="Handler for GET /v1.24/images/k8s.gcr.io/pause-amd64:3.1/json returned error: No such image: k8s.gcr.io/pause-amd64:3.1"
Apr 21 21:40:28 node1 dockerd[1144]: time="2018-04-21T21:40:28.515676456+08:00" level=error msg="Handler for GET /v1.24/images/k8s.gcr.io/pause-amd64:3.1/json returned error: No such image: k8s.gcr.io/pause-amd64:3.1"
Apr 21 21:40:28 node1 dockerd[1144]: time="2018-04-21T21:40:28.528631577+08:00" level=error msg="Handler for GET /v1.24/images/k8s.gcr.io/pause-amd64:3.1/json returned error: No such image: k8s.gcr.io/pause-amd64:3.1"
Apr 21 21:40:28 node1 dockerd[1144]: time="2018-04-21T21:40:28.535282319+08:00" level=error msg="Handler for GET /v1.24/images/k8s.gcr.io/pause-amd64:3.1/json returned error: No such image: k8s.gcr.io/pause-amd64:3.1"
Apr 21 21:40:28 node1 dockerd[1144]: time="2018-04-21T21:40:28.544880690+08:00" level=error msg="Handler for GET /v1.24/images/k8s.gcr.io/pause-amd64:3.1/json returned error: No such image: k8s.gcr.io/pause-amd64:3.1"
```

{% hint style="info" %}
这个窗口不要关，后面还有好几个镜像需要手动从国外下载。
{% endhint %}

回到刚才的`kubeadm init`窗口，按⌃c终止任务，然后通过如下命令找出需要下载的Docker镜像

```text
root@node1:/etc# grep image /etc/kubernetes/manifests/*
/etc/kubernetes/manifests/etcd.yaml:    image: k8s.gcr.io/etcd-amd64:3.1.12
/etc/kubernetes/manifests/kube-apiserver.yaml:    image: k8s.gcr.io/kube-apiserver-amd64:v1.10.1
/etc/kubernetes/manifests/kube-controller-manager.yaml:    image: k8s.gcr.io/kube-controller-manager-amd64:v1.10.1
/etc/kubernetes/manifests/kube-scheduler.yaml:    image: k8s.gcr.io/kube-scheduler-amd64:v1.10.1
```

目前，我们就找出了如下5个用到的镜像：

```text
k8s.gcr.io/pause-amd64:3.1
k8s.gcr.io/etcd-amd64:3.1.12
k8s.gcr.io/kube-apiserver-amd64:v1.10.1
k8s.gcr.io/kube-controller-manager-amd64:v1.10.1
k8s.gcr.io/kube-scheduler-amd64:v1.10.1
```

现在，找一台国外的服务器，加载上面的镜像，推送到国内自己的Gitlab Docker Registry

```bash
# 从国外下载镜像

images=(
k8s.gcr.io/pause-amd64:3.1
k8s.gcr.io/etcd-amd64:3.1.12
k8s.gcr.io/kube-apiserver-amd64:v1.10.1
k8s.gcr.io/kube-controller-manager-amd64:v1.10.1
k8s.gcr.io/kube-scheduler-amd64:v1.10.1
k8s.gcr.io/kube-proxy-amd64:v1.10.1
)

for sourceImageName in ${images[@]} ; do
    targetImageName=docker.corp.tanmer.com/tanmer/dockers/${sourceImageName}
    (    docker pull ${sourceImageName} \
      && docker tag ${sourceImageName} ${targetImageName} \
      && docker push ${targetImageName} \
      && docker rmi ${targetImageName} \
    ) || break
done
```

现在，我们回到刚才要安装`kubeadm init`的服务器，执行下面命令，下载镜像

```bash
# 恢复镜像到国内

images=(
k8s.gcr.io/pause-amd64:3.1
k8s.gcr.io/etcd-amd64:3.1.12
k8s.gcr.io/kube-apiserver-amd64:v1.10.1
k8s.gcr.io/kube-controller-manager-amd64:v1.10.1
k8s.gcr.io/kube-scheduler-amd64:v1.10.1
k8s.gcr.io/kube-proxy-amd64:v1.10.1
)

for targetImageName in ${images[@]} ; do
    sourceImageName=docker.corp.tanmer.com/tanmer/dockers/${targetImageName}
    (    docker pull ${sourceImageName} \
      && docker tag ${sourceImageName} ${targetImageName} \
      && docker rmi ${sourceImageName} \
    ) || break
done
```

### 正式初始化集群

```bash
# 重置刚才的集群命令
kubeadm reset
# 重新初始化
kubeadm init --pod-network-cidr=10.244.0.0/16
```

{% hint style="info" %}
这里用到了参数`--pod-network-cidr=10.244.0.0/16` 是因为我们使用`Flannel`网络，会在后面配置。
{% endhint %}

初始化成功之后，输入如下：

```text
root@node1:~# kubeadm init --pod-network-cidr=10.244.0.0/16
[init] Using Kubernetes version: v1.10.1
[init] Using Authorization modes: [Node RBAC]
[preflight] Running pre-flight checks.
	[WARNING FileExisting-crictl]: crictl not found in system path
Suggestion: go get github.com/kubernetes-incubator/cri-tools/cmd/crictl
[preflight] Starting the kubelet service
[certificates] Generated ca certificate and key.
[certificates] Generated apiserver certificate and key.
[certificates] apiserver serving cert is signed for DNS names [node1 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.0.1.10]
[certificates] Generated apiserver-kubelet-client certificate and key.
[certificates] Generated etcd/ca certificate and key.
[certificates] Generated etcd/server certificate and key.
[certificates] etcd/server serving cert is signed for DNS names [localhost] and IPs [127.0.0.1]
[certificates] Generated etcd/peer certificate and key.
[certificates] etcd/peer serving cert is signed for DNS names [node1] and IPs [10.0.1.10]
[certificates] Generated etcd/healthcheck-client certificate and key.
[certificates] Generated apiserver-etcd-client certificate and key.
[certificates] Generated sa key and public key.
[certificates] Generated front-proxy-ca certificate and key.
[certificates] Generated front-proxy-client certificate and key.
[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
[controlplane] Wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
[controlplane] Wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
[controlplane] Wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
[etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
[init] Waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests".
[init] This might take a minute or longer if the control plane images have to be pulled.
[apiclient] All control plane components are healthy after 21.002070 seconds
[uploadconfig] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[markmaster] Will mark node node1 as master by adding a label and a taint
[markmaster] Master node1 tainted and labelled with key/value: node-role.kubernetes.io/master=""
[bootstraptoken] Using token: sxs6qp.neypsgvrkx4whmqm
[bootstraptoken] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: kube-dns
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 10.0.1.10:6443 --token sxs6qp.neypsgvrkx4whmqm --discovery-token-ca-cert-hash sha256:242ca9d395b945ca774b28dbfc617f250152ae0a9b700c55be682f6f35a53edd

```

根据输出提示，我们拷贝kube配置文件，测试通过`kubectl`获取集群状态。

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

xiaohui@node1:~$ kubectl cluster-info
Kubernetes master is running at https://10.0.1.10:6443
KubeDNS is running at https://10.0.1.10:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

## 加入worker节点

### 安装Docker、kubeadm和kubectl

登录`node2`服务器，按照上面步骤安装`Docker`、`kubeadm`和`kubectl`

执行Master节点的输出提示：

```bash
kubeadm join 10.0.1.10:6443 --token sxs6qp.neypsgvrkx4whmqm --discovery-token-ca-cert-hash sha256:242ca9d395b945ca774b28dbfc617f250152ae0a9b700c55be682f6f35a53edd
```

这里也可以用journalctl -u docker -f查看日志，知道哪些镜像下载失败。

### 下载镜像

```bash
images=(
k8s.gcr.io/pause-amd64:3.1
k8s.gcr.io/kube-proxy-amd64:v1.10.1
)

for targetImageName in ${images[@]} ; do
    sourceImageName=docker.corp.tanmer.com/tanmer/dockers/${targetImageName}
    (    docker pull ${sourceImageName} \
      && docker tag ${sourceImageName} ${targetImageName} \
      && docker rmi ${sourceImageName} \
    ) || break
done
```

## 安装Flannel网络

现在，我们回到Master节点，查看节点状态，会发现他们的状态都是NotReady，这是因为我们还没有安装网络插件。

{% code-tabs %}
{% code-tabs-item title="On node1" %}
```bash
xiaohui@node1:~$ kubectl get no
NAME      STATUS     ROLES     AGE       VERSION
node1     NotReady   master    23m       v1.10.1
node2     NotReady   <none>    5m        v1.10.1
```
{% endcode-tabs-item %}
{% endcode-tabs %}

继续翻墙下载镜像

```bash
# 从国外下载镜像

images=(
k8s.gcr.io/k8s-dns-sidecar-amd64:1.14.8
k8s.gcr.io/k8s-dns-dnsmasq-nanny-amd64:1.14.8
k8s.gcr.io/k8s-dns-kube-dns-amd64:1.14.8
)

for sourceImageName in ${images[@]} ; do
    targetImageName=docker.corp.tanmer.com/tanmer/dockers/${sourceImageName}
    (    docker pull ${sourceImageName} \
      && docker tag ${sourceImageName} ${targetImageName} \
      && docker push ${targetImageName} \
      && docker rmi ${targetImageName} \
    ) || break
done

# 恢复镜像到国内

images=(
k8s.gcr.io/k8s-dns-sidecar-amd64:1.14.8
k8s.gcr.io/k8s-dns-dnsmasq-nanny-amd64:1.14.8
k8s.gcr.io/k8s-dns-kube-dns-amd64:1.14.8
)

for targetImageName in ${images[@]} ; do
    sourceImageName=docker.corp.tanmer.com/tanmer/dockers/${targetImageName}
    (    docker pull ${sourceImageName} \
      && docker tag ${sourceImageName} ${targetImageName} \
      && docker rmi ${sourceImageName} \
    ) || break
done

# 从国外下载镜像

docker pull quay.io/coreos/flannel:v0.9.1-amd64
docker tag quay.io/coreos/flannel:v0.9.1-amd64 docker.corp.tanmer.com/tanmer/dockers/flannel:v0.9.1-amd64
docker push docker.corp.tanmer.com/tanmer/dockers/flannel:v0.9.1-amd64
docker rmi docker.corp.tanmer.com/tanmer/dockers/flannel:v0.9.1-amd64

# 恢复镜像到国内

docker pull docker.corp.tanmer.com/tanmer/dockers/flannel:v0.9.1-amd64
docker tag docker.corp.tanmer.com/tanmer/dockers/flannel:v0.9.1-amd64 quay.io/coreos/flannel:v0.9.1-amd64
docker rmi docker.corp.tanmer.com/tanmer/dockers/flannel:v0.9.1-amd64

```

现在查看集群状态，所有`STATUS`都变成了`Ready`

```bash
xiaohui@node1:~$ kubectl get no
NAME      STATUS    ROLES     AGE       VERSION
node1     Ready     master    51m       v1.10.1
node2     Ready     <none>    34m       v1.10.1

xiaohui@tanmer-dev:~$ kubectl get po --all-namespaces -o wide
NAMESPACE     NAME                            READY     STATUS    RESTARTS   AGE       IP           NODE
kube-system   etcd-node1                      1/1       Running   0          1h        10.0.1.10    node1
kube-system   kube-apiserver-node1            1/1       Running   0          1h        10.0.1.10    node1
kube-system   kube-controller-manager-node1   1/1       Running   0          1h        10.0.1.10    node1
kube-system   kube-dns-86f4d74b45-lhf58       3/3       Running   2          1h        10.244.0.3   node1
kube-system   kube-flannel-ds-fwd78           1/1       Running   0          34m       10.0.1.11    node2
kube-system   kube-flannel-ds-hpf5s           1/1       Running   0          34m       10.0.1.10    node1
kube-system   kube-proxy-4vf42                1/1       Running   0          43m       10.0.1.11    node2
kube-system   kube-proxy-kvv94                1/1       Running   0          1h        10.0.1.10    node1
kube-system   kube-scheduler-node1            1/1       Running   0          1h        10.0.1.10    node1
```

##  安装Dashboard

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

这里，又需要翻墙下载镜像 `k8s.gcr.io/kubernetes-dashboard-amd64:v1.8.3`

> 官方文档：[https://github.com/kubernetes/dashboard](https://github.com/kubernetes/dashboard)

Dashboard启动之后，通过下面命令，代理API端口

```bash
kubectl proxy
```

然后访问 [http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/](http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/)

![](../.gitbook/assets/image%20%2810%29.png)

### 需要创建一个帐号

这里需要创建一个帐号才能访问。

{% code-tabs %}
{% code-tabs-item title="admin-user.yaml" %}
```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```
{% endcode-tabs-item %}
{% endcode-tabs %}

```bash
kubectl apply -f admin-user.yaml
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

把输出的Token填入网站

![](../.gitbook/assets/image%20%283%29.png)

![](../.gitbook/assets/image%20%2824%29.png)

### 安装Heapster

> 官方文档：[https://github.com/kubernetes/heapster](https://github.com/kubernetes/heapster)

这里又是翻墙下载镜像：

```text
k8s.gcr.io/heapster-influxdb-amd64:v1.3.3
k8s.gcr.io/heapster-amd64:v1.4.2
k8s.gcr.io/heapster-grafana-amd64:v4.4.3
```

```bash
# 安装influxdb
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb.yaml
# 安装heapster
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster.yaml
# 安装grafana
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/grafana.yaml
# 分配权限
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml
```

这里Grafana的内部端口是`3000`，如果需要外网访问，可以修改一下deploy配置，设置密码保护，然后通过Ingress绑定域名实现外网访问。

现在能够看到，`Heapster`和`monitoring-grafana`已经安装成功

![](../.gitbook/assets/image%20%289%29.png)

Grafana service by default requests for a LoadBalancer. If that is not available in your cluster, consider changing that to NodePort. Use the external IP assigned to the Grafana service, to access Grafana. The default user name and password is 'admin'. Once you login to Grafana, add a datasource that is InfluxDB. The URL for InfluxDB will be `http://INFLUXDB_HOST:INFLUXDB_PORT`. Database name is 'k8s'. Default user name and password is 'root'.

Grafana访问地址：

```text
http://localhost:8001/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
```

现在，Dashboard就能看到服务器的资源使用情况了

![](../.gitbook/assets/image%20%2813%29.png)

## 安装Weave Scope

Weave Scope是一个Kubernetes的可视化监控工具，通过他可以总览整个集群构架，实时诊断容器。

安装非常简单，一个命令搞定，最重要的是不用翻墙：

```bash
kubectl apply -f "https://cloud.weave.works/k8s/scope.yaml?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

```text
xiaohui@node1:~$ kubectl -n weave get all
NAME                                   READY     STATUS    RESTARTS   AGE
pod/weave-scope-agent-lwxk9            1/1       Running   0          1m
pod/weave-scope-agent-mfdnc            1/1       Running   0          1m
pod/weave-scope-app-69f8c6745d-sfj75   1/1       Running   0          1m

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/weave-scope-app   ClusterIP   10.102.231.71   <none>        80/TCP    1m

NAME                               DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/weave-scope-agent   2         2         2         2            2           <none>          1m

NAME                              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/weave-scope-app   1         1         1            1           1m

NAME                                         DESIRED   CURRENT   READY     AGE
replicaset.apps/weave-scope-app-69f8c6745d   1         1         1         1m
```

映射Pod端口到本地`4040`端口

```bash
kubectl port-forward -n weave "$(kubectl get -n weave pod --selector=weave-scope-component=app -o jsonpath='{.items..metadata.name}')" 4040
```

本地浏览器访问 [http://localhost:4040](http://localhost:4040)

![](../.gitbook/assets/image%20%2812%29.png)

## 安装Ingress

要让服务能够被外部网络通过域名访问，我们需要安装Ingress。

官方文档：[https://kubernetes.io/docs/concepts/services-networking/ingress/](https://kubernetes.io/docs/concepts/services-networking/ingress/)

这里我们可以使用官方的[`ingress-nginx`](https://github.com/kubernetes/ingress-nginx/blob/master/README.md)或[`traefik`](https://traefik.io/)，这里我们选用功能更完善的`traefik`

### Traefik

![](../.gitbook/assets/image%20%288%29.png)

参考文档：

[https://github.com/rootsongjc/kubernetes-handbook/blob/master/practice/edge-node-configuration.md](https://github.com/rootsongjc/kubernetes-handbook/blob/master/practice/edge-node-configuration.md)

以下配置文件可以在[kubernetes-handbook](https://github.com/rootsongjc/kubernetes-handbook)GitHub仓库中的[../manifests/traefik-ingress/](https://github.com/rootsongjc/kubernetes-handbook/blob/master/manifests/traefik-ingress/)目录下找到。

#### 创建服务帐号

{% code-tabs %}
{% code-tabs-item title="ingress-rbac.yaml" %}
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ingress
  namespace: kube-system

---

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: ingress
subjects:
  - kind: ServiceAccount
    name: ingress
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```
{% endcode-tabs-item %}
{% endcode-tabs %}

```bash
kubectl apply -f ingress-rbac.yaml
```

#### 部署Traefik DaemonSet

{% code-tabs %}
{% code-tabs-item title="traefik.yaml" %}
```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: traefik-ingress-lb
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      terminationGracePeriodSeconds: 60
      hostNetwork: true
      restartPolicy: Always
      serviceAccountName: ingress
      containers:
      - image: traefik
        name: traefik-ingress-lb
        resources:
          limits:
            cpu: 200m
            memory: 30Mi
          requests:
            cpu: 100m
            memory: 20Mi
        ports:
        - name: http
          containerPort: 80
          hostPort: 80
        - name: admin
          containerPort: 8580
          hostPort: 8580
        args:
        - --web
        - --web.address=:8580
        - --kubernetes
      nodeSelector:
        edgenode: "true"
```
{% endcode-tabs-item %}
{% endcode-tabs %}

```bash
kubectl apply -f traefik.yaml
```

{% hint style="info" %}
注意：我们使用了nodeSelector选择边缘节点来调度traefik-ingress-lb运行在它上面，所有你需要使用：
{% endhint %}

```bash
kubectl label nodes node2 edgenode=true
```

可以看到，应用已经启动了

```bash
xiaohui@tanmer-dev:~$ kubectl -n kube-system get ds
NAME                 DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR                   AGE
kube-flannel-ds      2         2         2         2            2           beta.kubernetes.io/arch=amd64   3h
kube-proxy           2         2         2         2            2           <none>                          3h
traefik-ingress-lb   1         1         1         1            1           edgenode=true                   1m
```

现在开启Web端管理服务，域名`traefik-ui.local`指向UI

{% code-tabs %}
{% code-tabs-item title="traefik-ui.yaml" %}
```yaml
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
  - name: web
    port: 80
    targetPort: 8580
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  rules:
  - host: traefik-ui.local
    http:
      paths:
      - path: /
        backend:
          serviceName: traefik-web-ui
          servicePort: web
```
{% endcode-tabs-item %}
{% endcode-tabs %}

```bash
kubectl apply -f traefik-ui.yaml
```

本地电脑改一下/etc/hosts文件，指向node2就能访问traefik web UI了

![](../.gitbook/assets/image%20%2815%29.png)

### Ingress高可用

[kube-keepalived-vip](https://github.com/kubernetes/contrib/tree/master/keepalived-vip)通过一个VIP，自动在Ingress节点上漂移，实现高可用

通常的负载均衡DNS 轮询模式是这样的：

```text
                                                  ___________________
                                                 |                   |
                                           |-----| Host IP: 10.4.0.3 |
                                           |     |___________________|
                                           |
                                           |      ___________________
                                           |     |                   |
Public ----(example.com = 10.4.0.3/4/5)----|-----| Host IP: 10.4.0.4 |
                                           |     |___________________|
                                           |
                                           |      ___________________
                                           |     |                   |
                                           |-----| Host IP: 10.4.0.5 |
                                                 |___________________|
```

采用VIP之后，变成这样：

```text
                                               ___________________
                                              |                   |
                                              | VIP: 10.4.0.50    |
                                        |-----| Host IP: 10.4.0.3 |
                                        |     | Role: Master      |
                                        |     |___________________|
                                        |
                                        |      ___________________
                                        |     |                   |
                                        |     | VIP: Unassigned   |
Public ----(example.com = 10.4.0.50)----|-----| Host IP: 10.4.0.4 |
                                        |     | Role: Slave       |
                                        |     |___________________|
                                        |
                                        |      ___________________
                                        |     |                   |
                                        |     | VIP: Unassigned   |
                                        |-----| Host IP: 10.4.0.5 |
                                              | Role: Slave       |
                                              |___________________|
```

#### 开始安装

添加权限：

```bash
kubectl -n kube-system create sa kube-keepalived-vip

cat <<EOS|kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-keepalived-vip
rules:
- apiGroups: [""]
  resources:
  - pods
  - nodes
  - endpoints
  - services
  - configmaps
  verbs: ["get", "list", "watch"]
EOS

cat <<EOS|kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube-keepalived-vip
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-keepalived-vip
subjects:
- kind: ServiceAccount
  name: kube-keepalived-vip
  namespace: kube-system
EOS
```

#### 配置VIP指向的服务

```bash
# 创建服务
cat <<EOS|kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: traefik-lb
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
  - name: web
    port: 80
    targetPort: 80
EOS

cat <<EOS|kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: vip-configmap
  namespace: kube-system
data:
  10.0.1.2: kube-system/traefik-lb
EOS
```

#### 部署DaemonSet

```bash
cat <<EOS|kubectl apply -f -
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: kube-keepalived-vip
  namespace: kube-system
spec:
  template:
    metadata:
      labels:
        name: kube-keepalived-vip
    spec:
      hostNetwork: true
      serviceAccountName: kube-keepalived-vip
      containers:
        - image: k8s.gcr.io/kube-keepalived-vip:0.11
          name: kube-keepalived-vip
          imagePullPolicy: IfNotPresent
          securityContext:
            privileged: true
          volumeMounts:
            - mountPath: /lib/modules
              name: modules
              readOnly: true
            - mountPath: /dev
              name: dev
          # use downward API
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          # to use unicast
          args:
          - --services-configmap=kube-system/vip-configmap
          # unicast uses the ip of the nodes instead of multicast
          # this is useful if running in cloud providers (like AWS)
          #- --use-unicast=true
          # vrrp version can be set to 2.  Default 3.
          #- --vrrp-version=2
      volumes:
        - name: modules
          hostPath:
            path: /lib/modules
        - name: dev
          hostPath:
            path: /dev
      nodeSelector:
        edgenode: "true"
EOS
```

## 翻墙下载镜像汇总

```bash
从国外下载镜像

images=(
k8s.gcr.io/pause-amd64:3.1
k8s.gcr.io/etcd-amd64:3.1.12
k8s.gcr.io/kube-apiserver-amd64:v1.10.1
k8s.gcr.io/kube-controller-manager-amd64:v1.10.1
k8s.gcr.io/kube-scheduler-amd64:v1.10.1
k8s.gcr.io/kube-proxy-amd64:v1.10.1
k8s.gcr.io/k8s-dns-sidecar-amd64:1.14.8
k8s.gcr.io/k8s-dns-dnsmasq-nanny-amd64:1.14.8
k8s.gcr.io/k8s-dns-kube-dns-amd64:1.14.8
k8s.gcr.io/kubernetes-dashboard-amd64:v1.8.3
k8s.gcr.io/heapster-influxdb-amd64:v1.3.3
k8s.gcr.io/heapster-amd64:v1.4.2
k8s.gcr.io/heapster-grafana-amd64:v4.4.3
k8s.gcr.io/kube-keepalived-vip:0.11
)

for sourceImageName in ${images[@]} ; do
    targetImageName=docker.corp.tanmer.com/tanmer/dockers/${sourceImageName}
    (    docker pull ${sourceImageName} \
      && docker tag ${sourceImageName} ${targetImageName} \
      && docker push ${targetImageName} \
      && docker rmi ${targetImageName} \
    ) || break
done


恢复镜像到国内

for targetImageName in ${images[@]} ; do
    sourceImageName=docker.corp.tanmer.com/tanmer/dockers/${targetImageName}
    (    docker pull ${sourceImageName} \
      && docker tag ${sourceImageName} ${targetImageName} \
      && docker rmi ${sourceImageName} \
    ) || break
done
```
