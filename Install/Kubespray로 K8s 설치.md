# Kubespray로 K8s 설치

<br>

## 구성 환경
#### public
- Bastion

#### private
- Control-plane
- Worker-node-1
- Worker-node-2
- Worker-node-3

#### OS
- Bastion 및 모든 server는 ubuntu로 진행

<br>

## 사전 준비 사항
### ssh 설정
```
$ ssh-keygen -t ed25519
$ ls ~/.ssh/
authorized_keys  id_ed25519  id_ed25519.pub

$ cat ~/.ssh/id_ed25519.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBsJyyDnHPKCYPhnzLf+SBSmm2dxIyfxYAteH/To6gl3 ubuntu@ip-<bastion ip>

# control plane 및 worker nodes에 접속 후, 실행
$ echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBsJyyDnHPKCYPhnzLf+SBSmm2dxIyfxYAteH/To6gl3 ubuntu@ip-<bastion ip>" >> ~/.ssh/authorized_keys

# bastion에서 control plane 및 worker nodes에 ssh 접속
$ ssh <control plane ip>
$ ssh <worker node 1 ip>
$ ssh <worker node 2 ip>
$ ssh <worker node 3 ip>
```

### Python3 및 pip3 설치
```
$ sudo apt-get update
$ sudo apt-get -y install python3
$ sudo apt-get -y install python3-pip

$ python3 -V
Python 3.10.6

$ pip3 -V
pip 22.0.2 from /usr/lib/python3/dist-packages/pip (python 3.10)
```

### virtualenv 설치
격리된 Python 환경을 구성하는 도구.  
Python version 및 project 별 종속성 문제 해결 가능.

```
$ sudo apt-get -y install python3-virtualenv
```

### git 설치
```
sudo apt-get -y install git
```

<br>

## Ansible 설치
사용 가능한 Python version에 따라 사용할 ansible version 선택이 제한될 수 있음.  
kubespray에서 사용하는 ansible version을 Python 가상 환경에 배포하는 것을 권장.

```
git clone https://github.com/kubernetes-sigs/kubespray.git

VENVDIR=kubespray-venv
KUBESPRAYDIR=kubespray
virtualenv  --python=$(which python3) $VENVDIR
source $VENVDIR/bin/activate
cd $KUBESPRAYDIR

pip3 install -U -r requirements.txt
```

### Ansible Python 호환성
|Ansible Version|Python Version|
|---|---|
|2.11|2.7,3.5-3.9|
|2.12|3.8-3.10|

<br>

## Ansible 설정
```
cp -rfp inventory/sample inventory/mycluster

# inventory builder로 Ansible inventory file update
declare -a IPS=(<control plane ip> <worker node 1 ip> <worker node 2 ip> <worker node 3 ip>)
CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}

# hosts.yaml 수정
## control plane, node, etcd 등 설정
vi inventory/mycluster/hosts.yaml
---
all:
  hosts:
    node1:
      ansible_host: <control plane ip>
      ip: <control plane ip>
      access_ip: <control plane ip>
    node2:
      ansible_host: <worker node 1 ip>
      ip: <worker node 1 ip>
      access_ip: <worker node 1 ip>
    node3:
      ansible_host: <worker node 2 ip>
      ip: <worker node 2 ip>
      access_ip: <worker node 2 ip>
    node4:
      ansible_host: <worker node 3 ip>
      ip: <worker node 3 ip>
      access_ip: <worker node 3 ip>
  children:
    kube_control_plane:
      hosts:
        node1:
    kube_node:
      hosts:
        node2:
        node3:
        node4:
    etcd:
      hosts:
        node1:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
---

# Ansible Playbook으로 Kubespray 배포 - playbook을 root로 실행
# /etc/에 SSL keys writing, packages 설치 그리고 다양한 systemd daemons와 interacting 등과 같은 작업을 위해 `--become` option 필수.
# `--become` option이 없으면 playbook 실행 실패.
ansible-playbook -i inventory/mycluster/hosts.yaml  --become --become-user=root cluster.yml
```

<hr>

## 참고
- **kubespray** - https://github.com/kubernetes-sigs/kubespray
- **Ansible 설치** - https://github.com/kubernetes-sigs/kubespray/blob/master/docs/ansible.md#installing-ansible
