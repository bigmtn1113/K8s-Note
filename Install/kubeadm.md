# kubeadm 설치

<br>

## 확인 사항
- K8s project는 Debian 및 Red Hat 기반의 Linux 배포판과 package 관리자가 없는 배포판에 대한 일반 지침을 제공.
- Machines당 2GB 이상의 RAM(이보다 적으면 apps을 위한 공간이 거의 남지 않음).
- 2 CPUs 이상.
- Cluster에 있는 모든 machines 간의 전체 network 연결(공용 또는 사설 network가 양호).
- kubelet이 제대로 작동하려면 swap을 비활성화.
  - 임시: `sudo swapoff -a`
  - 지속: system에 구성된 방식에 따라 `/etc/fstab`, `systemd.swap`와 같은 구성 파일에서 swap이 비활성화되어 있는지 확인.  
    `free -m` 또는 `swapon -s`로 확인

### 모든 노드의 MAC 주소와 product_uuid가 고유한지 확인
- network interfaces의 MAC 주소 확인: `ip link` or `ifconfig -a`
- product_uuid 확인: `sudo cat /sys/class/dmi/id/product_uuid`

### 필수 ports 확인
K8s 구성 요소가 서로 통신하려면 특정 port open 필요.

#### Control plane
|Protocol|Direction|Port Range|Purpose|Used By|
|---|---|---|---|---|
|TCP|Inbound|6443|Kubernetes API server|All|
|TCP|Inbound|2379-2380|etcd server client API|kube-apiserver, etcd|
|TCP|Inbound|10250|Kubelet API|Self, Control plane|
|TCP|Inbound|10259|kube-scheduler|Self|
|TCP|Inbound|10257|kube-controller-manager|Self|

#### worker node(s)
|Protocol|Direction|Port Range|Purpose|Used By|
|---|---|---|---|---|
|TCP|Inbound|10250|Kubelet API|Self, Control plane|
|TCP|Inbound|30000-32767|NodePort Services|All|

<br>

## Container runtime 설치
Pods 안에서 containers를 실행시키기 위해 K8s는 container runtime을 사용.  
기본적으로 K8s는 선택된 container runtime과 통신하기 위해 CRI(Container Runtime Interface)를 사용.  
Runtime을 지정하지 않으면 kubeadm은 알려진 endpoints 목록을 scan하여 설치된 container runtime을 자동으로 감지하려고 시도.  
Container runtime이 여러 개이거나 전혀 감지되지 않으면 kubeadm은 오류를 발생시키고 사용할 것을 지정하도록 요청.

※ Docker Engine은 CRI를 구현하지 않으므로 추가 서비스인 cri-dockerd를 설치 필요.  
cri-dockerd는 version 1.24에서 kubelet에서 제거된 legacy 기본 제공 Docker Engine 지원을 기반으로 하는 project.

Linux에 대해 알려진 endpoints  
|Runtime|Unix domain socket 경로|
|---|---|
|containerd|unix:///var/run/containerd/containerd.sock|
|CRI-O|unix:///var/run/crio/crio.sock|
|Docker Engine (cri-dockerd 사용)|unix:///var/run/cri-dockerd.sock|

### 필수 요소들 설치 및 구성
Linux의 K8s nodes를 위한 일반적인 설정들을 적용.

#### IPv4를 forwarding하여 iptables가 bredge된 traffic 보게 하기
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

다음 명령을 실행하여 `br_netfilter`, `overlay` modules가 load되었는지 확인.
```
$ lsmod | grep br_netfilter
br_netfilter           36864  0
bridge                307200  1 br_netfilter

$ lsmod | grep overlay
overlay               167936  0
```

다음 명령을 실행하여 `sysctl` config에서 `net.bridge.bridge-nf-call-iptables`, `net.bridge.bridge-nf-call-ip6tables` 및 `net.ipv4.ip_forward` system 변수가 `1`로 설정되었는지 확인.
```
$ sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

### containerd 설치
containerd 뿐만 아니라 runc와 CNI plugins도 같이 download 필요.  
편의상 `sudo su -`후 진행.

#### 1. containerd 설치
```
$ curl -LO https://github.com/containerd/containerd/releases/download/v1.7.1/containerd-1.7.1-linux-amd64.tar.gz
$ tar Cxzvf /usr/local containerd-1.7.1-linux-amd64.tar.gz
bin/
bin/containerd-shim-runc-v2
bin/containerd-shim
bin/ctr
bin/containerd-shim-runc-v1
bin/containerd
bin/containerd-stress
```

※ systemd를 통해 containerd를 시작하는 경우
```
$ mkdir -p /usr/local/lib/systemd/system/
$ curl https://raw.githubusercontent.com/containerd/containerd/main/containerd.service > /usr/local/lib/systemd/system/containerd.service

$ systemctl daemon-reload
$ systemctl enable --now containerd
Created symlink /etc/systemd/system/multi-user.target.wants/containerd.service → /usr/local/lib/systemd/system/containerd.service.
```

#### 2. runc 설치
```
$ curl -LO https://github.com/opencontainers/runc/releases/download/v1.1.7/runc.amd64
$ install -m 755 runc.amd64 /usr/local/sbin/runc
```

#### 3. CNI plugins 설치
```
$ mkdir -p /opt/cni/bin
$ curl -LO https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
$ tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.3.0.tgz
./
./macvlan
./static
./vlan
./portmap
./host-local
./vrf
./bridge
./tuning
./firewall
./host-device
./sbr
./loopback
./dhcp
./ptp
./ipvlan
./bandwidth
```

<br>

## kubeadm, kubelet, kubectl 설치
모든 machines에 설치
- `kubeadm`: cluster를 bootstrap하는 명령어
- `kubelet`: cluster의 모든 machines에서 실행되고 pods 및 containers 시작과 같은 작업을 수행하는 구성 요소
- `kubectl`: cluster와 대화하기 위한 command line util

kubeadm `kubelet`이나 `kubectl`을 설치하거나 관리하지 않으므로 kubeadm이 설치하려는 K8s control plane의 version과 일치하는지 확인 필요.  
그렇지 않으면 예기치 않은 버그가 있는 동작으로 이어질 수 있는 version skew이 발생할 위험 존재.  
kubelet과 control plane 사이의 minor version skew는 지원되지만 kubelet version은 API server version을 초과할 수 없음.  
ex) 1.7.0을 실행하는 kubelet은 1.8.0 API server와 완전히 호환되어야 하지만 그 반대는 불가

### Debian 기반
#### 1. `apt` package index를 update하고 K8s `apt` repository를 사용하는 데 필요한 packages 설치.
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

#### 2. Google Cloud 공개 서명 키 download.
`sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg`

#### 3. K8s `apt` repository 추가.
`echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list`

#### 4. `apt` package index를 update하고 kubelet, kubeadm 및 kubectl을 설치하고 해당 version을 고정.
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

※ Debian 12 및 Ubuntu 22.04 이전 releases에는 기본적으로 `/etc/apt/keyrings`가 존재하지 않음.  
필요한 경우 이 directory를 생성하여 누구나 읽을 수 있지만 관리자만 쓸 수 있도록 생성 가능.

### Red Hat 기반
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```

`setenforce 0`을 실행하여 SELinux를 허용 모드로 설정하고 `sed ...`로 효과적으로 비활성화.  
이는 예를 들어 pod networks에 필요한 host filesystem에 containers가 access할 수 있도록 하는 데 필요.  
kubelet에서 SELinux 지원이 개선될 때까지 이 작업을 수행

구성 방법을 알고 있으면 SELinux를 활성화된 상태로 둘 수 있지만 kubeadm에서 지원하지 않는 설정이 필요할 수 있음.

`baseurl`이 실패하면 Red Hat 기반 배포판이 `basearch`을 해석할 수 없으므로 `\$basearch`를 computer의 architecture로 교체.  
해당 값을 보려면 `uname -m`을 입력.  
ex) `x86_64`의 `baseurl` URL: `https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64.`

<br>

## cgroup driver 구성
Container runtime과 kubelet은 "cgroup driver"라는 속성을 갖고 있으며, cgroup driver는 Linux machines의 cgroups의 관리 측면에 있어서 중요.  
Container runtime과 kubelet의 cgroup drivers를 일치시켜야 하며, 그렇지 않으면 kubelet process에 오류가 발생.

### Container runtime의 cgroup driver를 systemd로 설정
Containerd는 daemon 수준 options를 지정하기 위해 `/etc/containerd/config.toml`에 있는 구성 file을 사용.  
기본 구성은 `containerd config default > /etc/containerd/config.toml`을 통해 생성 가능.

```
mkdir -p /etc/containerd/
containerd config default > /etc/containerd/config.toml
sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml

systemctl restart containerd
```

SystemdCgroup이 true로 바뀌었는지 확인
```
containerd config dump | grep SystemdCgroup
```

<hr>

## 참고
- kubeadm 설치 - https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- containerd 설치 - https://github.com/containerd/containerd/blob/main/docs/getting-started.md
- K8s 필수 ports - https://kubernetes.io/docs/reference/networking/ports-and-protocols/
