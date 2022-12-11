---
layout: post
title: 'Kubernetes Quickstart'
categories:
- Quickstart
tags:
- Infra
- DevOps
- Kubernetes
---

## Setup System

### Check Iptables Backend

kubeadm은 nftables와 호환되지 않으므로 iptables가 nftables 백엔드를 사용하는지 확인해야한다. RHEL/CentOS 7 기준으로는 해당되지 않으며 RHEL/CentOS 8은 레거시 모드 전환을 지원하지 않으므로 현재 kubeadm 패키지와 호환되지 않는다.

### Open Iptables Port

각 노드들이 통신할 수 있도록 방화벽 설정을 해야한다.

#### Master

| Protocol | Direction | Port Range | Purpose                 | Used By              |
|----------|-----------|------------|-------------------------|----------------------|
| TCP      | Inbound   | 6443*      | Kubernetes API server   | All                  |
| TCP      | Inbound   | 2379-2380  | etcd server client API  | kube-apiserver, etcd |
| TCP      | Inbound   | 10250      | Kubelet API             | Self, Control plane  |
| TCP      | Inbound   | 10251      | kube-scheduler          | Self                 |
| TCP      | Inbound   | 10252      | kube-controller-manager | Self                 |

#### Node

| Protocol | Direction | Port Range  | Purpose             | Used By             |
|----------|-----------|-------------|---------------------|---------------------|
| TCP      | Inbound   | 10250       | Kubelet API         | Self, Control plane |
| TCP      | Inbound   | 30000-32767 | NodePort Services** | All                 |

#### Calico Port Requirements

| Configuration                                     | Host(s)             | Connection Type | Protocol                                             |
|---------------------------------------------------|---------------------|-----------------|------------------------------------------------------|
| Calico networking (BGP)                           | All                 | Bidirectional   | TCP 179                                              |
| Calico networking with IP-in-IP enabled (default) | All                 | Bidirectional   | IP-in-IP, often represented by its protocol number 4 |
| Calico networking with VXLAN enabled              | All                 | Bidirectional   | UDP 4789                                             |
| Calico networking with Typha enabled              | Typha agent hosts   | Incoming        | TCP 5473 (default)                                   |
| flannel networking (VXLAN)                        | All                 | Bidirectional   | UDP 4789                                             |
| All                                               | kube-apiserver host | Incoming        | Often TCP 443 or 6443*                               |
| etcd datastore                                    | etcd hosts          | Incoming        | Officially TCP 2379 but can vary                     |

#### Master

```
-A INPUT -p tcp -m state --state NEW -m tcp --dport 179 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 5473 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 6443 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 10250 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 10256 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 10251 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 10252 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 2379:2380 -j ACCEPT
-A INPUT -p udp -m state --state NEW -m udp --dport 8285 -j ACCEPT
-A INPUT -p udp -m state --state NEW -m udp --dport 8472 -j ACCEPT
-A INPUT -p udp -m state --state NEW -m udp --dport 4789 -j ACCEPT
```

#### Node

```
-A INPUT -p tcp -m state --state NEW -m tcp --dport 179 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 5473 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 10250 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 10256 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 2379:2380 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 30000:32767 -j ACCEPT
-A INPUT -p udp -m state --state NEW -m udp --dport 8285 -j ACCEPT
-A INPUT -p udp -m state --state NEW -m udp --dport 8472 -j ACCEPT
-A INPUT -p udp -m state --state NEW -m udp --dport 4789 -j ACCEPT
```

#### Remove `FOWARD REJECT` Rule from Iptables

Kubernetes 클러스터는 포워딩을 사용하므로 iptables에서 `FORWAD REJECT` 규칙을 제거해야한다.

```bash
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
```

### Disable SELinux

컨테이너들이 호스트 파일시스템에 접근할수 있도록 SELinux를 비활성화해아한다. SELinux에 대한 지원이 향상되기 전까지는 이 작업을 수행해야한다.

```bash
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

### Edit Kernel Parameter

RHEL/CentOS 7의 일부 사용자는 iptables가 무시되어 트래픽이 잘못 라우팅되는 경우가 있다고 하니 이와 관련된 커널 파라미터를 수정해준다.

```bash
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

### Install Container Runtime

v1.6.0부터 Kubernetes는 기본적으로 CRI(Container Runtime Interface)를 사용한다.

v1.14.0부터 kubeadm은 잘 알려진 도메인 소켓 목록을 스캔하여 Linux 노드에서 컨테이너 런타임을 자동으로 감지하려고 시도한다. Docker와 containerd가 함께 감지되면 Docker가 우선되며 Docker와 containerd 외에 다른 둘 이상의 런타임이 감지되면 kubeadm은 오류 메세지와 함께 종료된다.

Linux 이외의 노드에서 기본적으로 사용되는 컨테이너 런타임은 Docker이다.

#### Install Docker

```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce

mkdir -p /etc/docker/
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

systemctl daemon-reload
systemctl enable --now docker
```

### Install kubeadm, kubectl and kubelet

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable --now kubelet
```

## Setup Kubernetes HA Cluster

고가용성 Kubernetes 구성시에는 etcd 클러스터를 내부에 구성할수도 있고 외부 etcd 클러스터를 구성 후 연결하여 사용하는 것도 가능하다. 컨트롤 플레인 노드 내부에 etcd를 구성할 경우 정적 etcd 팟이 생성된다.

![kubeadm-ha-topology-stacked-etcd](/images/2020-05-14-kubernetes-quickstart/kubeadm-ha-topology-stacked-etcd.png)

![kubeadm-ha-topology-external-etcd](/images/2020-05-14-kubernetes-quickstart/kubeadm-ha-topology-external-etcd.png)

### Requirements

- 마스터 노드에 대한 kubeadm 최소요구사항을 충족하는 3대의 컴퓨터
- 워커 노드에 대한 kubeadm 최소요구사항을 충족하는 3대의 컴퓨터
- 클러스터 호스트 간의 네트워크 연결
- 모든 머신에서의 sudo 권한
- kubeadm, kubelet(kubectl은 선택사항)

### Setup Load balancer for kube-apiserver

kube-apiserver에 접근할 수 있도록 로드밸런서를 구성한다. kube-apiserver의 기본 포트는 6443이다.

직접 IP를 사용하기보단 도메인네임을 통해 구성하는 것이 좋다.

로드밸런서는 모든 컨트롤 플레인 노드의  API 서버와 통신할 수 있어야하며 수신 포트에서 들어오는 트래픽을 허용해야한다.

DSR을 사용한다면 VIP를 루프백에 추가해주고 ARP 관련 문제가 발생하지 않도록 커널 파라미터를 수정한다.

클러스터에 신규 마스터 노드를 참가시킬때 API 서버와의 통신이 필요하다. 루프백으로 자기자신을 바라보도록 설정이 되어있으면 아직 API 서버가 생성되지 않은 자기 자신에게 요청을 하게 되므로 클러스터 참가에 실패하게 된다. 첫번째 마스터 노드가 아닌 나머지 마스터 노드들은 클러스터에 참가 시킨 후 루프백을 추가해야한다.

```bash
cat >>  /etc/sysctl.conf  << EOF
##L4 setup
net.ipv4.conf.lo.arp_ignore = 1
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
EOF
sysctl -p
ip addr add ${API_SERVER_IP}/32 dev lo
```

### Initialize First Master

```bash
kubeadm init \
  --control-plane-endpoint "$API_SERVER_DNS:$API_SERVER_PORT" \
  --pod-network-cidr $POD_NETWORK_CIDR \
  --upload-certs
```

위에서 설정한 로드밸런서 설정에 맞춰 `API_SERVER_DNS`와 `API_SERVER_PORT`를 지정한다.

`--pod-network-cidr`의 경우 사용하려는 CNI 플러그인에 따라 설정한다. Calico는 기본적으로 `192.168.0.0/16` 대역을 사용한다.

초기화 과정이 끝나면 클러스터를 사용하기 위한 추가 명령어가 제공되며 다른 노드가 클러스터에 가입하기 위한 명령어도 출력된다.

```bash
....
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join master.example.com:6443 --token kvi39q.gc30umo57gmz9q3y \
    --discovery-token-ca-cert-hash sha256:e165f13dc76e2f5b1b5b2e5b44696bf4d4e20fded258f247dd6b523904db6c91 \
    --control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join master.example.com:6443 --token kvi39q.gc30umo57gmz9q3y \
    --discovery-token-ca-cert-hash sha256:e165f13dc76e2f5b1b5b2e5b44696bf4d4e20fded258f247dd6b523904db6c91
```

`$HOME/.kube` 디렉토리를 초기화하는 스크립트를 실행 후 join 명령어들을 따로 복사해둔다.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### Install CNI Plugin

pod끼리 통신할 수 있도록 네트워크 플러그인을 설치한다.

Calico의 기본적으로 `192.168.0.0/16` 대역을 팟 네트워크에 사용하며 스크립트에 이 값이 하드코딩되어있다. 만약 다른 네트워크를 사용한다면 스크립트에서 `CALICO_IPV4POOL_CIDR` 환경변수의 값을 `kubeadm init` 과정에서 사용한 `pod-network-cidr` 옵션 값과 동일하게 변경해주어야 한다.

```bash
curl https://docs.projectcalico.org/manifests/calico.yaml -O
sed -i -e "s?192.168.0.0/16?$POD_NETWORK_CIDR?g" calico.yaml
kubectl apply -f calico.yaml
```

```bash
configmap "calico-config" created
customresourcedefinition.apiextensions.k8s.io "felixconfigurations.crd.projectcalico.org" created
customresourcedefinition.apiextensions.k8s.io "ipamblocks.crd.projectcalico.org" created
customresourcedefinition.apiextensions.k8s.io "blockaffinities.crd.projectcalico.org" created
customresourcedefinition.apiextensions.k8s.io "ipamhandles.crd.projectcalico.org" created
customresourcedefinition.apiextensions.k8s.io "bgppeers.crd.projectcalico.org" created
customresourcedefinition.apiextensions.k8s.io "bgpconfigurations.crd.projectcalico.org" created
customresourcedefinition.apiextensions.k8s.io "ippools.crd.projectcalico.org" created
customresourcedefinition.apiextensions.k8s.io "hostendpoints.crd.projectcalico.org" created
customresourcedefinition.apiextensions.k8s.io "clusterinformations.crd.projectcalico.org" created
customresourcedefinition.apiextensions.k8s.io "globalnetworkpolicies.crd.projectcalico.org" created
customresourcedefinition.apiextensions.k8s.io "globalnetworksets.crd.projectcalico.org" created
customresourcedefinition.apiextensions.k8s.io "networksets.crd.projectcalico.org" created
customresourcedefinition.apiextensions.k8s.io "networkpolicies.crd.projectcalico.org" created
clusterrole.rbac.authorization.k8s.io "calico-kube-controllers" created
clusterrolebinding.rbac.authorization.k8s.io "calico-kube-controllers" created
clusterrole.rbac.authorization.k8s.io "calico-node" created
clusterrolebinding.rbac.authorization.k8s.io "calico-node" created
daemonset.extensions "calico-node" created
serviceaccount "calico-node" created
deployment.extensions "calico-kube-controllers" created
serviceaccount "calico-kube-controllers" created
```

아래 명령어를 통해 모든 팟이 정상적으로 실행되는지 확인한다.

```bash
watch kubectl get pods --all-namespaces
```

```bash
NAMESPACE    NAME                                       READY  STATUS   RESTARTS  AGE
kube-system  calico-kube-controllers-6ff88bf6d4-tgtzb   1/1    Running  0         2m45s
kube-system  calico-node-24h85                          1/1    Running  0         2m43s
kube-system  coredns-846jhw23g9-9af73                   1/1    Running  0         4m5s
kube-system  coredns-846jhw23g9-hmswk                   1/1    Running  0         4m5s
kube-system  etcd-jbaker-1                              1/1    Running  0         6m22s
kube-system  kube-apiserver-jbaker-1                    1/1    Running  0         6m12s
kube-system  kube-controller-manager-jbaker-1           1/1    Running  0         6m16s
kube-system  kube-proxy-8fzp2                           1/1    Running  0         5m16s
kube-system  kube-scheduler-jbaker-1                    1/1    Running  0         5m41s
```

#### Deploy Master Certifications

마스터 초기화시에 인증서를 업로드하도록 설정하지 않았을 경우 기본 컨트롤 플레인 노드에서 추가 제어 평면 노드로 인증서를 수동으로 복사해야한다.

```bash
USER=root
CONTROL_PLANE_IPS="kube02.example.com kube03.example.com"
ssh-keygen
for host in $CONTROL_PLANE_IPS; do
    ssh-copy-id "${USER}"@$host
    scp /etc/kubernetes/pki/ca.crt "${USER}"@$host:
    scp /etc/kubernetes/pki/ca.key "${USER}"@$host:
    scp /etc/kubernetes/pki/sa.key "${USER}"@$host:
    scp /etc/kubernetes/pki/sa.pub "${USER}"@$host:
    scp /etc/kubernetes/pki/front-proxy-ca.crt "${USER}"@$host:
    scp /etc/kubernetes/pki/front-proxy-ca.key "${USER}"@$host:
    scp /etc/kubernetes/pki/etcd/ca.crt "${USER}"@$host:etcd-ca.crt
    scp /etc/kubernetes/pki/etcd/ca.key "${USER}"@$host:etcd-ca.key
    ssh "${USER}"@$host 'mkdir -p /etc/kubernetes/pki/etcd'
    ssh "${USER}"@$host 'mv $HOME/ca.crt /etc/kubernetes/pki/'
    ssh "${USER}"@$host 'mv $HOME/ca.key /etc/kubernetes/pki/'
    ssh "${USER}"@$host 'mv $HOME/sa.pub /etc/kubernetes/pki/'
    ssh "${USER}"@$host 'mv $HOME/sa.key /etc/kubernetes/pki/'
    ssh "${USER}"@$host 'mv $HOME/front-proxy-ca.crt /etc/kubernetes/pki/'
    ssh "${USER}"@$host 'mv $HOME/front-proxy-ca.key /etc/kubernetes/pki/'
    ssh "${USER}"@$host 'mv $HOME/etcd-ca.crt /etc/kubernetes/pki/etcd/ca.crt'
    ssh "${USER}"@$host 'mv $HOME/etcd-ca.key /etc/kubernetes/pki/etcd/ca.key'
done
```

### Initialize Other Master

나머지 마스터 노드에서 이전 마스터 노드 초기화시에 출력된 컨트롤 플래인 노드 추가 명령어를 실행한다.

```bash
kubeadm join master.example.com:6443 --token kvi39q.gc30umo57gmz9q3y \
    --discovery-token-ca-cert-hash sha256:e165f13dc76e2f5b1b5b2e5b44696bf4d4e20fded258f247dd6b523904db6c91 \
    --control-plane
```

```bash
....
This node has joined the cluster and a new control plane instance was created:

* Certificate signing request was sent to apiserver and approval was received.
* The Kubelet was informed of the new secure connection details.
* Control plane (master) label and taint were applied to the new node.
* The Kubernetes control plane instances scaled up.
* A new etcd member was added to the local/stacked etcd cluster.

To start administering your cluster from this node, you need to run the following as a regular user:

	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster.
```

작업이 종료되면 추가 명령어를 수행한다.

```bash
	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Initialize Node

기존에 복사해두었던 join 커맨드를 실행한다.

```bash
kubeadm join master.example.com:6443 --token kvi39q.gc30umo57gmz9q3y \
    --discovery-token-ca-cert-hash sha256:e165f13dc76e2f5b1b5b2e5b44696bf4d4e20fded258f247dd6b523904db6c91
```

## Using kubectl

kubectl은 Kubernetes 클러스터를 관리하기 위한 커맨드라인 도구이다.

### Install kubectl

#### Linux

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
yum install -y kubectl
```

#### macOS

```bash
brew install kubectl
```

#### Windows

[최신버전 바이너리 다운로드 링크](https://storage.googleapis.com/kubernetes-release/release/v1.17.0/bin/windows/amd64/kubectl.exe)

### Connect to Remote Kubernetes Cluster

로컬에서 kubectl을 사용하기 위해서는 마스터 노드에서 `.kube` 설정 디렉토리를 로컬로 복사해야한다. 기존에 OpenShift를 포함해 다른 Kubernetes 클러스터를 사용했을 경우 이미 설정 디렉토리가 존재할 수 있으니 확인 후 백업해둬야한다.

```bash
ssh "$LOCAL_PC_USER@$LOCAL_PC_IP" "mkdir -p ~/.kube"
scp -r $HOME/.kube/config "$LOCAL_PC_USER@$LOCAL_PC_IP:~/.kube.config":
```

윈도우의 경우 다음과 같은 위치에 복사한다.

```bash
cd %USERPROFILE%
mkdir .kube
```

### Check kubectl

아래와 같이 `kubectl version` 명령어를 통해 서버와 클라이언트의 버전 정보를 확인할 수 있어야한다.

```bash
kubectl version
Client Version: version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.2", GitCommit:"59603c6e503c87169aea6106f57b9f242f64df89", GitTreeState:"clean", BuildDate:"2020-01-18T23:30:10Z", GoVersion:"go1.13.5", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.2", GitCommit:"59603c6e503c87169aea6106f57b9f242f64df89", GitTreeState:"clean", BuildDate:"2020-01-18T23:22:30Z", GoVersion:"go1.13.5", Compiler:"gc", Platform:"linux/amd64"}
```

## Kubernetes Dashboard

### Install Kubernetes Dashboard

웹상에서 Kubernetes 클러스터를 관리하기 위해 대시보드를 설치한다.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
```

### Add API Certification to Local Browser

PC에서 Kubernetes API 서버를 통해 대시보드에 접속하기 위해서는 PC의 브라우저에 인증서를 추가해주어야한다.

클러스터 설정을 통해 인증서 파일을 생성한다.

```bash
grep 'client-certificate-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt
grep 'client-key-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key
openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -out kubecfg.p12 -name "kubernetes-client"
```

생성된 `kubecfg.p12` 파일을 PC로 복사하여 각 브라우저 지침에 맞게 신뢰할 수 있는 인증서로 추가해준다.

### Create Service Account

대시보드 로그인을 위해 계정을 생성하고 충분한 권한을 부여한다.

```bash
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
EOF

cat <<EOF | kubectl create -f -
apiVersion: rbac.authorization.k8s.io/v1
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
EOF
```

아래 명령어를 실행하여 생성된 계정에 대한 토큰을 확인한다.

```bash
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

### Access Kubernetes Dashboard via API Server

`API_SERVER_DNS`와 `API_SERVICE_PORT`를 적절히 지정하여 아래 주소로 접근한다.

로그인은 이전에 확인한 토큰을 통해 진행한다.

```bash
https://API_SERVER_DNS:API_SERVER_PORT/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/overview?namespace=default
```

## Using Helm 3

Helm은 Kubernetes를 위한 패키지 매니저이다. Kubernetes 애플리케이션의 구성이나 설치, 업그레이드를 쉽게할 수 있도록 도와준다.

Helm에는 세 가지 큰 개념이 존재한다.

- 차트(Chart)에는 Kubernetes 클러스터 내에서 애플리케이션이나 도구, 서비스를 실행하는데 필요한 모든 리소스 정의가 포함되어 있다.
- 저장소(Repository)는 차트를 수집하고 공유할 수 있는 장소이다.
- 릴리즈(Release)는 Kubernetes 클러스터에서 실행되는 차트의 인스턴스이다. 하나의 차트로 클러스터 내에서 여러 개의 인스턴스를 설치할 수 있다.

Helm 2에서는 tiller 서버를 사용하였으나 보안상 문제점이 있기 때문에 Helm 3에서는 tiller 서버 없이 사용하게 되어있다.

Helm을 사용하기 위해서는 시스템에 kubectl이 설치되어 있어야하며 클러스터에 연결 가능한 상태여야한다.

### Install Helm

#### From the bianary releases

[Helm 릴리즈 페이지](https://github.com/helm/helm/releases)에서 Helm 바이너리를 다운로드 받을 수 있다. 아키텍쳐에 맞는 바이너리를 다운로드 받아 바로 사용할 수 있다.

```bash
tar -zxvf helm-v3.0.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
```

#### From Homebrew(macOS)

```bash
brew install helm
```

#### From Chocolatey(Windows)

```bash
choco install kubernetes-helm
```

## Monitoring Kubernetes with Prometheus

Prometheus는 오픈소스 모니터링 솔루션이다.

키/값 쌍으로 이루어진 시계열 데이터를 저장하는 다차원 데이터 모델을 사용한다.

데이터를 수집할 각 서버에서 Prometheus 서버로 데이터를 전송하는 방식(push)이 아니라 Prometheus 서버에서 각 서버로부터 데이터를 가져오는(pull) 방식을 사용한다.

Kubernetes 클러스터 모니터링에도 활용할 수 있으며 현시점에서 가장 주목받는 모니터링 도구 중 하나이다.

### Install Prometheus via Helm Chart

Kubernetes 애플리케이션 관리도구인 Helm을 이용하여 프로메테우스를 쉽게 구성할 수 있다.

우선 Helm Chart 저장소를 내려받고 Prometheus 차트 디렉토리로 이동한다.

```bash
git clone https://github.com/helm/charts.git
cd charts/stable/prometheus
```

프로메테우스는 데이터를 저장하기위해 PVC를 사용하므로 이에 대한 설정이 필요하나 데이터를 지속적으로 보관할 필요가 없다면 emptyDir을 통해 간단하게 배포할 수 있다.

에디터로 `values.yaml` 파일을 열고 `persistentVolume:` 항목을 찾아 `enabled: true`라고 되어 있는 부분을 `enabled: false`로 전부 바꾼다. alertmanager, prometheus server, push gateway에 대한 항목이 존재한다.

`values.yaml` 파일의 설정이 끝났다면 모니터링을 위한 네임스페이스를 생성하고 Prometheus를 설치한다.

```bash
kubectl create ns monitoring
helm install prometheus . -n monitoring
```

아래 명령어를 통해 모든 팟이 정상적으로 실행되었는지 확인한다.

```bash
watch kubectl get po -n monitoring
```

모든 팟이 정상적으로 생성되었다면 prometheus-server 팟의 정확한 이름을 확인한다.

```bash
kubectl get po -n monitoring                                                                                     1 ↵  43.20 Dur   master ●
NAME                                             READY   STATUS    RESTARTS   AGE
prometheus-alertmanager-88745986c-9vcsf          2/2     Running   0          79s
prometheus-kube-state-metrics-59bb448977-nnxgc   1/1     Running   0          79s
prometheus-node-exporter-9qnd2                   1/1     Running   0          79s
prometheus-node-exporter-tw4w7                   1/1     Running   0          79s
prometheus-pushgateway-646d94ffdc-x6v2l          1/1     Running   0          79s
prometheus-server-7d47688f57-g9tgh               2/2     Running   0          79s
```

Prometheus 서버는 9090 포트로 웹서비스를 제공하므로 9090 포트를 포워딩하고 `[http://localhost:9090](http://localhost:9090)` 주소로  Prometheus 서버에 접속해본다. 로컬에서 Cockpit 등의 서비스때문에 이미 9090 포트를 사용중이라면 `9099:9090`처럼 다른 로컬 포트를 사용하여 포워딩해도 된다.

```bash
kubectl port-forward -n monitoring prometheus-server-7d47688f57-g9tgh 9090:9090
```

### Grafana

Grafana는 오픈소스 시각화 도구이다. 데이터 소스로부터 데이터를 받아 해당 데이터들을 시각화할 수 있다. Promethues, InfluxDB, ElasticSearch 등 다양한 데이터 소스를 지원한다.

Prometheus도 자체적으로 그래프를 제공하긴하나 전문적인 시각화 도구에 비해서는 미흡한 편이므로 Grafana를 통해 데이터를 시각화하는 것이 좋다.

#### Install Grafana via Helm Chart

기존에 내려받은 Helm Charts 저장소에는 Grafana를 설치하기 위한 차트도 존재하므로 `charts/stable/grafana` 경로로 이동하여 설치를 진행하면 된다.

설치 전 `values.yaml`에 `adminPassword` 설정이 있으니 해당 항목을 입력한 뒤 설치를 진행한다.

```bash
helm install grafana . -n monitoring
```

Grafana 팟이 정상적으로 실행되었는지 확인한다.

```bash
watch kubectl get po -n monitoring
```

Grafana가 정상적으로 실행되었다면 포트 포워딩을 통해 Grafana에 접속한다. Grafana는 `3000` 포트로 웹서비스를 제공한다.

```bash
kubectl port-forward -n monitoring grafana-6d97f8bf67-96w6z 3000
```

Grafana 웹서비스에 접근했다면 이제 데이터 소스로 Prometheus를 연결하고 대시보드를 추가하면 된다.

#### Add Promethues Data Source

1. `Add data source` 선택

    ![Screenshot_from_2020-01-30_16-59-30](/images/2020-05-14-kubernetes-quickstart/Screenshot_from_2020-01-30_16-59-30.png)

2. `Prometheus`를 선택

    ![Screenshot_from_2020-01-30_17-00-31](/images/2020-05-14-kubernetes-quickstart/Screenshot_from_2020-01-30_17-00-31.png)

3. URL에 `http://prometheus-server.monitoring.svc.cluster.local` 입력 후 저장

    ![Screenshot_from_2020-01-30_17-06-54](/images/2020-05-14-kubernetes-quickstart/Screenshot_from_2020-01-30_17-06-54.png)

4. [https://grafana.com](https://grafana.com/grafana/dashboards?search=kubernetes)에서 적당한 Kubernetes Prometheus용 대시보드를 선택하여 ID를 복사한다.

    ![Screenshot_from_2020-01-30_17-18-25](/images/2020-05-14-kubernetes-quickstart/Screenshot_from_2020-01-30_17-18-25.png)

5. 네비게이션 메뉴에서 `Import` 선택

    ![Screenshot_from_2020-01-30_17-15-51](/images/2020-05-14-kubernetes-quickstart/Screenshot_from_2020-01-30_17-15-51.png)

6. 복사한 대시보드 ID 입력하여 대시보드 `Import` 선택

    ![Screenshot_from_2020-01-30_17-15-16](/images/2020-05-14-kubernetes-quickstart/Screenshot_from_2020-01-30_17-15-16.png)

    ![Screenshot_from_2020-01-30_17-14-06](/images/2020-05-14-kubernetes-quickstart/Screenshot_from_2020-01-30_17-14-06.png)

7. 대시보드 목록에서 추가한 대시보드를 확인

    ![Screenshot_from_2020-01-30_17-21-55](/images/2020-05-14-kubernetes-quickstart/Screenshot_from_2020-01-30_17-21-55.png)

## Expose Service via Ingress Nginx Controller

Kubernetes 클러스터 내의 서비스에 대한 외부 접근을 관리하는 API 오브젝트이며 일반적으로 HTTP를 관리한다. Ingress는 부하 분산, SSL 종료, 명칭 기반 가상 호스팅을 제공할 수 있다.

Ingress는 Kubernetes 클러스터 외부에서 클러스터 내부 서비스로 HTTP와 HTTPS 경로를 노출한다. 트래픽 라우팅은 Ingress 리소스에 정의된 규칙에 의해 컨트롤된다.

Ingress는 임의의 포트 또는 프로토콜을 노출시키지 않는다. HTTP(S) 외에 다른 서비스를 클러스터 외부로 노출하기위해서는 `Service.Type=NodePort` 또는 `Service.Type=LoadBalancer` 유형의 서비스를 사용한다.

이러한 규칙들을 지정해놓은 리소스를 **Ingress Resource**라고 하며 실제로 이 규칙들을 실행하는 것이 **Ingress Controller**이다. Ingress Controller는 Kubernetes 클러스터 구성시 자동으로 설치되지 않으며 필요에 따라 원하는 구현체를 클러스터 내에 배포해야한다.

### Install Ingress Nginx

Ingress Nginx를 배포하기 위해서는 우선 기반 스크립트를 적용해야한다.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.28.0/deploy/static/mandatory.yaml
```

이후 환경에 맞는 배포 스크립트를 적용한다.

#### Bare-metal

베어메탈 환경에서는 NordPort를 통해 Ingress Controller를 서비스한다.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.28.0/deploy/static/provider/baremetal/service-nodeport.yaml
```

### Setup External IP for External Load Balancer

외부 로드밸런서를 이용한다면 외부 IP 설정을 통해 편리하게 서비스들을 프록싱 할 수 있다.

아래 명령어로 기존에 NodePort로 설정된 Ingress Controller를 수정한다.

```bash
kubectl -n ingress-nginx edit svc ingress-nginx
```

설정파일 내에 존재하는 NodePort 관련 설정들을 삭제한다. `spec.type` 설정과 `spec.ports[*].nodePort` 설정을 제거하면 된다.

`spec.type: ClusterIP`로 변경하고 `spec.externalIPs: [VIP]` 설정을 추가한다.

설정이 끝나면 각 워커 노드에서 kube-proxy가 VIP:80, VIP:443 포트를 바인딩한 것을 확인할 수 있으며 [http://VIP](http://vip)로 접속해보면 웹서비스에 접근할 수 있다.

### Example: Grafana Ingress Resource

외부에서 Grafana에 접속할 수 있도록 Grafana Ingress를 설정한다.

```bash
cat << 'EOF' > grafana-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: grafana-ingress
spec:
  rules:
    - host: grafana.example.com
      http:
        paths:
          - path: /
            backend:
              serviceName: grafana
              servicePort: 80
EOF
kubectl create -f grafana-ingress.yaml -n monitoring
```

이제 [grafana.example.com](http://grafana.example.com)을 통해 Grafana에 접근할 수 있다.
