Kubeadm 설치

요구 사항
- K8s 프로젝트는 Debian 및 Red Hat 기반의 Linux 배포판과 패키지 관리자가 없는 배포판에 대한 일반 지침을 제공.
- Machines당 2GB 이상의 RAM(이보다 적으면 앱을 위한 공간이 거의 남지 않음).
- 2 CPUs 이상.
- 클러스터에 있는 모든 시스템 간의 전체 네트워크 연결(공용 또는 사설 네트워크가 양호함).
- kubelet이 제대로 작동하려면 스왑을 비활성화.
  - 임시: `sudo swapoff -a`
   - 지속: 시스템에서 구성된 방식에 따라 `/etc/fstab`, `systemd.swap`와 같은 구성 파일에서 스왑이 비활성화되어 있는지 확인

모든 노드의 MAC 주소와 product_uuid가 고유한지 확인
- 네트워크 인터페이스의 MAC 주소 확인: `ip link` or `ifconfig -a`
- product_uuid 확인: `sudo cat /sys/class/dmi/id/product_uuid`

필수 port 확인
- K8s 구성 요소가 서로 통신하려면 특정 port open 필요.
- ex) nc 127.0.0.1 6443

Container runtime 설치
- Pods 안에서 containers를 실행시키기 위해 K8s는 container runtime을 사용.
- 기본적으로 K8s는 선택된 container runtime과 통신하기 위해 CRI(Container Runtime Interface)를 사용.
- Runtime을 지정하지 않으면 kubeadm은 알려진 endpoints 목록을 scan하여 설치된 container runtime을 자동으로 감지하려고 시도.
- Container runtime이 여러 개이거나 전혀 감지되지 않으면 kubeadm은 오류를 발생시키고 사용할 것을 지정하도록 요청.

※ Docker Engine은 CRI를 구현하지 않으므로 추가 서비스인 cri-dockerd를 설치 필요.
- cri-dockerd는 version 1.24에서 kubelet에서 제거된 legacy 기본 제공 Docker Engine 지원을 기반으로 하는 project.

Linux에 대해 알려진 endpoints(Runtime - Unix domain socket 경로)
- containerd - unix:///var/run/containerd/containerd.sock
- CRI-O - unix:///var/run/crio/crio.sock
- Docker Engine (cri-dockerd 사용) - unix:///var/run/cri-dockerd.sock

kubeadm, kubelet, kubectl 설치
- 모든 machines에 설치
- kubeadm: cluster를 bootstrap하는 명령어
- kubelet: cluster의 모든 machines에서 실행되고 pods 및 containers 시작과 같은 작업을 수행하는 구성 요소
- kubectl: cluster와 대화하기 위한 command line util

kubeadm은 `kubelet`이나 `kubectl`을 설치하거나 관리하지 않으므로 kubeadm이 설치하려는 K8s control plane의 version과 일치하는지 확인 필요.
그렇지 않으면 예기치 않은 버그가 있는 동작으로 이어질 수 있는 version skew이 발생할 위험 존재.
kubelet과 control plane 사이의 minor version skew는 지원되지만 kubelet version은 API server version을 초과할 수 없음.
ex) 1.7.0을 실행하는 kubelet은 1.8.0 API server와 완전히 호환되어야 하지만 그 반대는 불가

- Debian 기반
1. `apt` package index를 update하고 K8s `apt` repository를 사용하는 데 필요한 packages 설치.
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

2. Google Cloud 공개 서명 키 download.
`sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg`

3. K8s `apt` repository 추가.
`echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list`

4. `apt` package index를 update하고 kubelet, kubeadm 및 kubectl을 설치하고 해당 version을 고정.
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

※ Debian 12 및 Ubuntu 22.04 이전 releases에는 기본적으로 `/etc/apt/keyrings`가 존재하지 않음.
필요한 경우 이 directory를 생성하여 누구나 읽을 수 있지만 관리자만 쓸 수 있도록 생성 가능.

- Red Hat 기반
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

`baseurl`이 실패하면 Red Hat 기반 배포판이 `basearch`을 해석할 수 없으므로 `\$basearch`를 computer의 architecture로 교체. 해당 값을 보려면 `uname -m`을 입력.
ex) `x86_64`의 `baseurl` URL: `https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64.`

---

kubeadm으로 cluster 생성
1. Control plane node 초기화
Control plane node는 control plane 구성 요소(etcd 및 API Server)가 실행되는 machine.

(1) (권장) 단일 control plane `kubeadm` cluster를 고가용성으로 upgrade할 계획이 있는 경우 모든 control plane nodes에 대한 공유 endpoint를 설정하도록 `--control-plane-endpoint` 지정 필요. 이러한 endpoint는 LB의 DNS 이름 또는 IP 주소일 수 있음.
(2) Pod network add-on을 선택하고 `kubeadm init`에 전달되는 인수가 필요한지 확인. 선택한 3rd-party 공급자에 따라 `--pod-network-cidr`를 공급자별 값으로 설정해야 할 수도 있음.
(3) (선택 사항) `kubeadm`은 잘 알려진 endpoints 목록을 사용하여 container runtime 감지 시도. 다른 container runtime을 사용하거나 provisioning된 node에 둘 이상 설치된 경우 `kubeadm`에 `--cri-socket` 대한 인수 지정.
(4) (선택 사항) 달리 지정하지 않는 한 `kubeadm`은 기본 gateway와 연결된 network interface를 사용하여 이 특정 control plane node의 API Server에 대한 advertise 주소를 설정. 다른 network interface를 사용하려면 `--apiserver-advertise-address=<ip-address>` 인수를 `kubeadm init`에 지정.

Control plane node 초기화
`kubeadm init <args>`

apiserver-advertise-address 및 ControlPlaneEndpoint에 대한 고려 사항
`--apiserver-advertise-address`이 특정 control plane node의 API Server에 대한 advertise 주소를 설정하는 데 사용될 수 있지만, `--control-plane-endpoint`은 모든 control plane nodes에 대한 공유 endpoint를 설정하는 데 사용 가능.
`--control-plane-endpoint`는 IP 주소에 mapping할 수 있는 IP 주소와 DNS 이름을 모두 허용.

Mapping example
`192.168.0.102 cluster-endpoint`

여기서 `192.168.0.102`이 node의 IP 주소고 `cluster-endpoint`는 이 IP에 mapping되는 사용자 지정 DNS 이름입니다. 이렇게 하면 `--control-plane-endpoint=cluster-endpoint`를 `kubeadm init`로 전달하고 동일한 DNS 이름을 `kubeadm join`에 전달 가능. 나중에 고가용성 scenario에서 LB의 주소를 가리키도록 `cluster-endpoint` 수정 가능.

`--control-plane-endpoint` 없이 생성된 단일 control plane cluster를 고가용성 클러스터로 전환하는 것은 kubeadm에서 지원하지 않음.

kubeadm init
`kubeadm init`는 먼저 machine이 K8s를 실행할 준비가 되었는지 확인하기 위해 일련의 사전 검사를 실행. 이러한 사전 검사는 경고를 표시하고 오류가 발생하면 종료. `kubeadm init`은 그런 다음 cluster control plane 구성 요소를 download하고 설치 진행.

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

Root가 아닌 사용자에 대해 kubectl이 작동하도록 하려면 `kubeadm init` 출력의 일부이기도 한 다음 명령을 실행.
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Cluster에 nodes를 join하려면 `kubeadm init`이 출력하는 `kubeadm join` 명령어가 필요하니 기록 필수.

Token은 control plane node와 joining nodes 간의 상호 인증에 사용됨. 여기에 포함된 token은 secret. 이 token이 있으면 누구나 인증된 nodes를 cluster에 추가할 수 있으므로 안전하게 보관 필요. 이러한 tokens은 `kubeadm token` 명령을 사용하여 나열, 생성 및 삭제 가능.

Pod network add-on 설치
Pods가 서로 통신할 수 있도록 CNI 기반 Pod network add-on 배포 필요. Cluster DNS(CoreDNS)는 network가 설치되기 전에 시작되지 않음.

Control plane node 또는 kubeconfig 자격 증명이 있는 node에서 다음 명령을 사용하여 Pod network add-on 설치 가능.
`kubectl apply -f <add-on.yaml>`

Cluster당 하나의 Pod network만 설치 가능.
Pod network가 설치되면 `kubectl get pods --all-namespaces` 출력 중 CoreDNS Pod가 `Running`인지 확인하여 작동하는지 확인 가능. 그리고 CoreDNS Pod가 가동되고 실행되면 nodes를 joining하여 계속 사용 가능.

Calico 설치
1. Tigera Calico operator 및 CRD(Custom resource definitions) 설치.
`kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/tigera-operator.yaml`

CRD bundle의 size가 크기 때문에, `kubectl apply`은 request limits를 초과할 수 있으므로 `kubectl create` 또는 `kubectl replace`를 사용.

2. 필요한 CR(Custom Resource)을 작성하여 Calico 설치.
`kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/custom-resources.yaml`

이 manifest를 만들기 전에 contents를 읽고 설정이 환경에 맞는지 확인 필요. 예를 들어 pod network CIDR과 일치하도록 기본 IP pool CIDR을 변경해야 할 수 있음.

3. 다음 명령을 사용하여 모든 Pods가 실행 중인지 확인.
`watch kubectl get pods -n calico-system`

Tigera operator는 `calico-system` namespace에 resources를 설치.

Nodes join
Nodes는 workloads(containers 및 Pods 등)가 실행되는 위치. Cluster에 새 nodes를 추가하려면 각 machine에 대해 다음을 수행.
- machine에 대한 SSH
- Root로 접근(e.g. `sudo su -`)
- 필요한 경우 runtime 설치
- `kubeadm init`에서 출력한 명령 실행.
  `kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>`

Token이 없는 경우, control plane node에서 다음 명령을 실행하여 얻을 수 있음.
`kubeadm token list`

출력은 다음과 유사.

```
TOKEN                    TTL  EXPIRES              USAGES           DESCRIPTION            EXTRA GROUPS
8ewj1p.9r9hcjoqgajrj4gi  23h  2018-06-12T02:51:28Z authentication,  The default bootstrap  system:
                                                   signing          token generated by     bootstrappers:
                                                                    'kubeadm init'.        kubeadm:
                                                                                           default-node-token
```

기본적으로 tokens는 24시간 후에 만료. 현재 token이 만료된 후 node를 cluster에 가입시키는 경우, control plane node에서 다음 명령을 실행하여 새 token 생성 가능.
`kubeadm token create`

출력은 다음과 유사.
`5didvk.d09sbcov8ph2amjw`

`--discovery-token-ca-cert-hash`의 값이 없는 경우, control plane node에서 다음 명령 chain을 실행하여 가져올 수 있음.

```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
```

출력은 다음과 유사.
`8cb2de97839780a412b93877f8507ad6c94f73add17d5d7058e91741c9d5ec78`


참고
- kubeadm 설치 - https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- kubeadm으로 cluster 생성 - https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
- calico 설치 - https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart
