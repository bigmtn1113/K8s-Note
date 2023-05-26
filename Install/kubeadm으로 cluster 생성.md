# kubeadm으로 cluster 생성

<br>

## 확인 사항
- deb/rpm 호환 Linux OS를 실행하는 하나 이상의 machines. ex) Ubuntu 또는 CentOS.
- Machine당 2GiB 이상의 RAM - 이보다 적으면 apps을 위한 공간이 거의 남지 않음.
- Control plane node로 사용하는 machine의 CPUs는 최소 2개 이상.
- Cluster의 모든 machines 간에 완전한 network 연결. 공용 또는 사설 network 사용 가능.

`kubeadm` 또한 새 cluster에서 사용하려는 K8s version을 배포할 수 있는 version 사용 필요. 이 page는 K8s v1.27용으로 작성.  
`kubeadm` 도구의 전반적인 기능 상태는 GA(General Availability).

<br>

## Control plane node 초기화
Control plane node는 control plane 구성 요소(etcd 및 API Server)가 실행되는 machine.

1. **(권장)** 단일 control plane `kubeadm` cluster를 고가용성으로 upgrade할 계획이 있는 경우 모든 control plane nodes에 대한 공유 endpoint를 설정하도록 `--control-plane-endpoint` 지정 필요. 이러한 endpoint는 LB의 DNS 이름 또는 IP 주소일 수 있음.
2. Pod network add-on을 선택하고 `kubeadm init`에 전달되는 인수가 필요한지 확인. 선택한 3rd-party 공급자에 따라 `--pod-network-cidr`를 공급자별 값으로 설정해야 할 수도 있음.

    ex) calico
    ```
    --pod-network-cidr=192.168.0.0/16
    ```
4. **(선택 사항)** `kubeadm`은 잘 알려진 endpoints 목록을 사용하여 container runtime 감지 시도. 다른 container runtime을 사용하거나 provisioning된 node에 둘 이상 설치된 경우 `kubeadm`에 `--cri-socket` 대한 인수 지정.

    ex) crio
    ```
    --cri-socket=/var/run/crio/crio.sock
    ```
5. **(선택 사항)** 달리 지정하지 않는 한 `kubeadm`은 기본 gateway와 연결된 network interface를 사용하여 이 특정 control plane node의 API Server에 대한 advertise 주소를 설정. 다른 network interface를 사용하려면 `--apiserver-advertise-address=<ip-address>` 인수를 `kubeadm init`에 지정.

    ex) master node(172.31.0.100)
    ```
    --apiserver-advertise-address 172.31.0.100
    ```

#### Control plane node 초기화
```
kubeadm init <args>
```

<br>

## apiserver-advertise-address 및 ControlPlaneEndpoint에 대한 고려 사항
`--apiserver-advertise-address`이 특정 control plane node의 API Server에 대한 advertise 주소를 설정하는 데 사용될 수 있지만, `--control-plane-endpoint`은 모든 control plane nodes에 대한 공유 endpoint를 설정하는 데 사용 가능.
`--control-plane-endpoint`는 IP 주소에 mapping할 수 있는 IP 주소와 DNS 이름을 모두 허용.

#### Mapping example
```
192.168.0.102 cluster-endpoint
```

여기서 `192.168.0.102`이 node의 IP 주소고 `cluster-endpoint`는 이 IP에 mapping되는 사용자 지정 DNS 이름.  
이렇게 하면 `--control-plane-endpoint=cluster-endpoint`를 `kubeadm init`로 전달하고 동일한 DNS 이름을 `kubeadm join`에 전달 가능.  
나중에 고가용성 scenario에서 LB의 주소를 가리키도록 `cluster-endpoint` 수정 가능.

`--control-plane-endpoint` 없이 생성된 단일 control plane cluster를 고가용성 클러스터로 전환하는 것은 kubeadm에서 지원하지 않음.

<br>

## kubeadm init
`kubeadm init`는 먼저 machine이 K8s를 실행할 준비가 되었는지 확인하기 위해 일련의 사전 검사를 실행.  
이러한 사전 검사는 경고를 표시하고 오류가 발생하면 종료.  
`kubeadm init`은 그런 다음 cluster control plane 구성 요소를 download하고 설치 진행.

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

Token은 control plane node와 joining nodes 간의 상호 인증에 사용됨. 여기에 포함된 token은 secret.  
이 token이 있으면 누구나 인증된 nodes를 cluster에 추가할 수 있으므로 안전하게 보관 필요.  
이러한 tokens은 `kubeadm token` 명령을 사용하여 나열, 생성 및 삭제 가능.

<br>

## Pod network add-on 설치
Pods가 서로 통신할 수 있도록 CNI 기반 Pod network add-on 배포 필요. Cluster DNS(CoreDNS)는 network가 설치되기 전에 시작되지 않음.

Control plane node 또는 kubeconfig 자격 증명이 있는 node에서 다음 명령을 사용하여 Pod network add-on 설치 가능.
```
kubectl apply -f <add-on.yaml>
```

Cluster당 하나의 Pod network만 설치 가능.  
Pod network가 설치되면 `kubectl get pods --all-namespaces` 출력 중 CoreDNS Pod가 `Running`인지 확인하여 작동하는지 확인 가능.  
그리고 CoreDNS Pod가 가동되고 실행되면 nodes를 joining하여 계속 사용 가능.

### Calico 설치
1. Tigera Calico operator 및 CRD(Custom resource definitions) 설치.
    ```
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/tigera-operator.yaml
    ```

    CRD bundle의 size가 크기 때문에, `kubectl apply`은 request limits를 초과할 수 있으므로 `kubectl create` 또는 `kubectl replace`를 사용.

2. 필요한 CR(Custom Resource)을 작성하여 Calico 설치.
     ```
     kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/custom-resources.yaml
     ```

    이 manifest를 만들기 전에 contents를 읽고 설정이 환경에 맞는지 확인 필요.  
    예를 들어 pod network CIDR과 일치하도록 기본 IP pool CIDR을 변경해야 할 수 있음.

3. 다음 명령을 사용하여 모든 Pods가 실행 중인지 확인.
    ```
    watch kubectl get pods -n calico-system
    ```

    Tigera operator는 `calico-system` namespace에 resources를 설치.

<br>

## Nodes join
Nodes는 workloads(containers 및 Pods 등)가 실행되는 위치. Cluster에 새 nodes를 추가하려면 각 machine에 대해 다음을 수행.
- machine에 대한 SSH
- Root로 접근(e.g. `sudo su -`)
- 필요한 경우 runtime 설치
- `kubeadm init`에서 출력한 명령 실행.
  ```
  kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>
  ```

Token이 없는 경우, control plane node에서 다음 명령을 실행하여 얻을 수 있음.
```
kubeadm token list
```

출력은 다음과 유사.
```
TOKEN                    TTL  EXPIRES              USAGES           DESCRIPTION            EXTRA GROUPS
8ewj1p.9r9hcjoqgajrj4gi  23h  2018-06-12T02:51:28Z authentication,  The default bootstrap  system:
                                                   signing          token generated by     bootstrappers:
                                                                    'kubeadm init'.        kubeadm:
                                                                                           default-node-token
```

기본적으로 tokens는 24시간 후에 만료.  
현재 token이 만료된 후 node를 cluster에 가입시키는 경우, control plane node에서 다음 명령을 실행하여 새 token 생성 가능.
```
kubeadm token create
```

출력은 다음과 유사.
```
5didvk.d09sbcov8ph2amjw
```

`--discovery-token-ca-cert-hash`의 값이 없는 경우, control plane node에서 다음 명령 chain을 실행하여 가져올 수 있음.
```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
```

출력은 다음과 유사.
```
8cb2de97839780a412b93877f8507ad6c94f73add17d5d7058e91741c9d5ec78
```

<hr>

## 참고
- kubeadm으로 cluster 생성 - https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
- calico 설치 - https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart
