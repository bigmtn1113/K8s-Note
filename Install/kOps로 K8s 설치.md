# kOps로 K8s 설치

<br>

## 요구 사항
- kubectl 설치
- 64-bit (AMD64, Intel 64) device architecture 기반
- AWS 계정, IAM keys 생성 및 구성. IAM 사용자는 다음과 같은 적절한 권한 필요.
  - AmazonEC2FullAccess
  - AmazonRoute53FullAccess
  - AmazonS3FullAccess
  - IAMFullAccess
  - AmazonVPCFullAccess
  - AmazonSQSFullAccess
  - AmazonEventBridgeFullAccess

<br>

## Cluster 구축
### 1. kOps 설치
Release page에서 kOps download

```
# Latest release download
curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64

# ex) kOps version v1.20.0 download
## curl -LO https://github.com/kubernetes/kops/releases/download/v1.20.0/kops-linux-amd64

chmod +x kops-linux-amd64

sudo mv kops-linux-amd64 /usr/local/bin/kops

```

### 2. Cluster에 사용할 route53 domain 생성
kOps는 cluster 내부와 외부 모두에서 검색을 위해 DNS을 사용하기에 clients에서 K8s API server에 연결 가능.  
이런 cluster 이름에 kOps는 명확한 견해을 가지는데: 반드시 유효한 DNS 이름이어야 함. 이렇게 함으로써 사용자는 clusters를 헷갈리지 않을것이고, 동료들과 혼선없이 공유할 수 있으며, IP를 기억할 필요없이 접근 가능.

Clusters를 구분하기 위해 subdomains을 활용 가능.  
ex) `useast1.dev.example.com`을 이용한다면, API server endpoint는 `api.useast1.dev.example.com`

```
aws route53 create-hosted-zone --name dev.example.com --caller-reference 1

dig NS dev.example.com
```

### 3. Clusters state 저장용 S3 bucket 생성
kOps는 설치 이후에도 clusters 관리 가능.  
이를 위해 사용자가 생성한 clusters의 상태나 사용하는 keys 정보들을 지속적으로 추적 필요. 이 정보를 S3에 저장. 이 bucket의 접근은 S3 권한으로 제어.

다수의 clusters는 동일한 S3 bucket을 이용할 수 있고, 사용자는 이 S3 bucket을 같은 clusters를 운영하는 동료에게 공유 가능.  
하지만 이 S3 bucket에 접근 가능한 사람은 사용자의 모든 clusters에 관리자 접근이 가능하게 되니, 운영팀 이외로 공유되지 않도록 주의 필요.  
그래서 보통 한 운영팀 당 하나의 S3 bucket을 관리.

- `AWS_PROFILE` 선언(AWS CLI 동작을 위해 다른 profile을 선택해야 할 경우).
- `aws s3 mb s3://clusters.dev.example.com`를 이용해 S3 버킷 생성.
- `export KOPS_STATE_STORE=s3://clusters.dev.example.com`하면, kOps는 이 위치를 기본값으로 인식. 이 부분을 bash profile등에 넣어두는것을 권장.

### 4. 클러스터 설정 구성
```
kops create cluster --zones=us-east-1c useast1.dev.example.com
```

kOps는 cluster에 사용될 설정을 생성.  
여기서 주의할 점은 실제 cloud resources가 아닌 configuration만을 생성한다는 것.

명령어 실행 후 출력되는 commands.
- Clusters 조회: `kops get cluster`
- Cluster 수정: `kops edit cluster useast1.dev.example.com`
- Instance group 수정: `kops edit ig --name=useast1.dev.example.com nodes`
- Master instance group 수정: `kops edit ig --name=useast1.dev.example.com master-us-east-1c`

※ 출력되는 commands는 상이할 수 있음.

Instance group은 K8s nodes로 등록된 instances의 집합이고 AWS상에서는 auto-scaling-groups를 통해 생성됨.  
사용자는 여러 개의 instance groups 관리 가능.

### 5. AWS에 cluster 생성
```
kops update cluster useast1.dev.example.com --yes
```

언제든 `kops update cluster`로 cluster 설정 변경 가능.  
`--yes`를 명시하지 않으면 `kops update cluster` 후 어떤 설정이 변경될지가 표시. 운영계 clusters 관리할 때 용이.

※ `kops rolling-update cluster`로 설정 원복 가능.

### 6. 확인
`kops update cluster` 명령 실행 후 출력되는 commands 확인.

- Cluster 검증: `kops validate cluster --wait 10m`
- Nodes 조회: `kubectl get nodes --show-labels`

<br>

## Cluster 삭제
```
kops delete cluster useast1.dev.example.com --yes
```

<br>

## ※ Troubleshooting
### `kops validate cluster` 및 `kubectl get nodes` error
#### 문제
```
$ kops validate cluster --wait 10m
...
W0531 06:05:04.257912    4552 validate_cluster.go:184] (will retry): unexpected error during validation: error listing nodes: Unauthorized
...
```
```
$ kubectl get nodes
...
E0531 06:07:40.410817    4678 memcache.go:265] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp 127.0.0.1:8080: connect: connection refused
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

#### 원인
`kubectl config view` 명령 실행 시 user 관련 설정이 되어 있지 않은 상태임을 확인.

```
$ kubectl config view
...
contexts:
- context:
    cluster: useast1.dev.example.com
    user: ""
  name: useast1.dev.example.com
...
users: null
```

#### 해결
```
$ kops export kubecfg --admin
```
```
$ kubectl config view
...
contexts:
- context:
    cluster: useast1.dev.example.com
    user: useast1.dev.example.com
  name: useast1.dev.example.com
...
users:
- name: useast1.dev.example.com
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
```
```
$ kops validate cluster --wait 10m
...
INSTANCE GROUPS
NAME                            ROLE            MACHINETYPE     MIN     MAX     SUBNETS
control-plane-ap-northeast-2a   ControlPlane    t3.medium       1       1       ap-northeast-2a
nodes-ap-northeast-2a           Node            t3.medium       3       3       ap-northeast-2a,ap-northeast-2c

NODE STATUS
NAME                    ROLE            READY
i-0471e6dd25752dc4a     node            True
i-083cf8abb4e02c91a     node            True
i-0d5162edc4223606d     node            True
i-0d5bf35a478a1cdab     control-plane   True

Your cluster kops.bigmtn.link is read
```
```
$ kubectl get node
NAME                  STATUS   ROLES           AGE     VERSION
i-0471e6dd25752dc4a   Ready    node            6m14s   v1.26.5
i-083cf8abb4e02c91a   Ready    node            6m41s   v1.26.5
i-0d5162edc4223606d   Ready    node            22m     v1.26.5
i-0d5bf35a478a1cdab   Ready    control-plane   24m     v1.26.5
```

<hr>

## 참고
- kOps로 K8s 설치 - https://kubernetes.io/docs/setup/production-environment/tools/kops/
- IAM user 권한 - https://github.com/kubernetes/kops/blob/master/docs/getting_started/aws.md#setup-iam-user
- kOps release page - https://github.com/kubernetes/kops/releases
