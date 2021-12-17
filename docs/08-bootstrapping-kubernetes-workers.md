# Bootstrapping the Kubernetes Worker Nodes

이번 구성은 Kubernete worker node들을 부트스트랩 하겠습니다. 각 노드에 다음 구성요소를 설치하겠습니다.
CNI의 경우는 calico를 설치할 것이며, 설치는 [Provisioning Pod Network CNI Calico](09-provisioning-pod-network-cni-calico.md)에서 진행하도록 하겠습니다.

## Prerequisites

이번 실습 명령어는 각 Worker node에서 진행해야 합니다. 그래서 각 node에 ssh로 접속해서 진행합니다.
Worker node: `k8s-worker-1`, `k8s-worker-2`, `k8s-worker-3`

```
ssh k8s-worker-1
```

### Running commands in parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki)를 사용하여 동시에 여러 node에 같은 명령어르 실행할 수 있습니다.  이 실습을 진행하기 전에 [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux)를 보시면 좋습니다.

## Provisioning a Kubernetes Worker Node

kubernetes Repository 추가:

```
cat << EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```

### Install the Kubernetes kubeadm, kubelet, kubectl

Kubernetes 설치:

```
sudo dnf -y install kubeadm-1.22.4 kubelet-1.22.4 kubectl-1.22.4 --disableexcludes=kubernetes
```

### Start kubelet

kubelet 실행

```
sudo systemctl enable kubelet
sudo systemctl start kubelet
```

### Join kubernetes cluster

`k8s-controller-1`에서 나오 정보를 가지고 Worker 노드들을 조인합니다.

```
sudo kubeadm join 192.168.56.10:6443 --token j202s7.tmu57rpu89n8k3f2 \
--discovery-token-ca-cert-hash sha256:fed9660155ad16378ebaa0ec09f91f2cb61dea93c19ca837a41e10210b79ec21
```

## Verification

Kubernetes nodes 등록 상태 확인:

`k8s-controller-1`에서 확인

```
  kubectl get nodes
```

> output

```
NAME               STATUS     ROLES                  AGE     VERSION
k8s-controller-1   NotReady   control-plane,master   36m     v1.22.4
k8s-controller-2   NotReady   control-plane,master   12m     v1.22.4
k8s-controller-3   NotReady   control-plane,master   11m     v1.22.4
k8s-worker-1       NotReady   <none>                 3m29s   v1.22.4
k8s-worker-2       NotReady   <none>                 3m29s   v1.22.4
k8s-worker-3       NotReady   <none>                 3m28s   v1.22.4
```

아직 네트워크 CNI를 설정하지 않았기 때문에 NotReady 상태로 표시 됩니다.

Next: [Provisioning Pod Network CNI Calico](09-provisioning-pod-network-cni-calico.md)
