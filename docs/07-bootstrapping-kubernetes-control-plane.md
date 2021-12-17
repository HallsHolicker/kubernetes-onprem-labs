# Bootstrapping the Kubernetes Control Plane

이번 구성은 고가용성을 위해 Kubernetes control plane을 3개의 node에 부트스트랩 합니다.

## Prerequisites

이번 실습 명령어는 각 Controller node에서 진행해야 합니다. 그래서 각 node에 ssh로 접속해서 진행합니다.
Controller node: `k8s-controller-1`, `k8s-controller-2`, `k8s-controller-3`

```
ssh k8s-controller-1
```

### Running commands in parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki)를 사용하여 동시에 여러 node에 같은 명령어르 실행할 수 있습니다.  이 실습을 진행하기 전에 [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux)를 보시면 좋습니다.

## The Kubernetes Frontend Load Balancer

이번 섹션은 Kubernetes API Server의 고가용성을 위해 `Keepalived` + `Haproxy`를 이용해서 endpoint에 사용되는 VIP 및 Load balancer를 설정하겠습니다.
Kubernetes Hardway를 진행하기 위한 방법이므로, `keepalived`와 `Haproxy`를 소스 설치가 아닌 dnf를 이용해서 설치하도록 하겠습니다.


### Install keepalived

Keepalived 설치:

```
sudo dnf -y install openssl-devel libnl3-devel keepalived
```

keepalived config 생성:

```
cat <<EOF | sudo tee /etc/keepalived/keepalived.conf
global_defs {
  @k8s-contoller-1 router_id k8s-contoller-1
  @k8s-contoller-2 router_id k8s-contoller-2
  @k8s-contoller-3 router_id k8s-contoller-3
  vrrp_version 3
}

vrrp_script chk_haproxy {
  script "/sbin/pidof haproxy"
  interval 5
  weight 2
}

vrrp_instance HAProxy {
  interface enp0s8
  virtual_router_id 51
  @k8s-contoller-1 state MASTER
  @k8s-contoller-2 state BACKUP
  @k8s-contoller-3 state BACKUP

  @k8s-contoller-1 priority 101
  @k8s-contoller-2 priority 100
  @k8s-contoller-3 priority 99
  
  @k8s-contoller-1 unicast_src_ip 192.168.56.11
  @k8s-contoller-2 unicast_src_ip 192.168.56.12
  @k8s-contoller-3 unicast_src_ip 192.168.56.13

  advert_int 1
  authentication {
    auth_type PASS
    auth_pass k8s-hardway
  }
  unicast_peer {
  
    @^k8s-contoller-1 unicast_src_ip 192.168.56.11
    @^k8s-contoller-2 unicast_src_ip 192.168.56.12
    @^k8s-contoller-3 unicast_src_ip 192.168.56.13
  }
  virtual_ipaddress {
    $(grep "k8s-controller$" /etc/hosts | awk '{print $1}')
  }
  track_script{
    chk_haproxy
  }
}
EOF
```

### Verification keepalived

keepalived 실행:

```
sudo systemctl enable keepalived
sudo systemctl start keepalived
```
> output

```
ip a show dev enp0s8
```

```
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:95:0a:3d brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.11/24 brd 10.240.0.255 scope global noprefixroute enp0s8
       valid_lft forever preferred_lft forever
    inet 192.168.56.10/32 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe95:a3d/64 scope link
       valid_lft forever preferred_lft forever
```
> output의 192.168.56.10은 `k8s-controller-1` 에만 보이게 됩니다.

### Install Haproxy

Haproxy 설치:

```
sudo dnf -y install make gcc gcc-c++ pcre-devel haproxy
```

### Setting Haproxy

Haproxy config 설정:

```
cat <<EOF | sudo tee /etc/haproxy/haproxy.cfg
global
  daemon
  user haproxy
  group haproxy
  master-worker
  maxconn 10240
  stats socket /var/run/haproxy.sock mode 660 level admin
  stats timeout 60s

  tune.ssl.default-dh-param 2048

frontend k8s-contoller
  bind 192.168.56.10:80
  bind 127.0.0.1:80
  mode http
  option tcplog
  default_backend k8s-controller

backend k8s-controller
  mode http
  option tcp-check
  balance roundrobin
  default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
  server k8s-controller-1 192.168.56.11:6443 check ssl verify none
  server k8s-controller-2 192.168.56.12:6443 check ssl verify none
  server k8s-controller-3 192.168.56.13:6443 check ssl verify none
EOF
```

Haproxy가 bind IP가 없어도 실행되도록 하기 위해 커널값을 수정:

```
sudo sysctl -w net.ipv4.ip_nonlocal_bind=1
```

Haproxy 실행:

```
sudo systemctl enable haproxy
sudo systemctl start haproxy
```

### Verification Haproxy

Haproxy 정상 동작 확인

```
curl -H "Host: kubernetes.default.svc.cluster.local" -i http://127.0.0.1/healthz
```

아직 kubernetes API server를 실행하지 않았기에 503 에러가 나옵니다. Kubernetes API Server까지 설정을 완료하고 다시 확인을 하도록 하겠습니다.

```
HTTP/1.0 503 Service Unavailable
Cache-Control: no-cache
Connection: close
Content-Type: text/html

<html><body><h1>503 Service Unavailable</h1>
No server is available to handle this request.
</body></html>
```


## Provision the Kubernetes Control Plane

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

### Install private CA

사설 CA를 이용한 kubernetes 구성을 위해 기존 [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)에서 만든 파일을 kubernetes pki 위치로 이동

```

sudo mkdir /etc/kubernetes/pki
sudo cp ca.crt /etc/kubernetes/pki
sudo cp ca.key /etc/kubernetes/pki

```

### Create kubeadm-config.yml

Controller 및 Worker의 등록/삭제 및 외부 ETCD를 사용하기 위해 kubeadm-config.yml을 만듭니다.
kubernetes 연동에 사용될 token 및 certificate-key는 테스트 랩 구성을 위해서 임시로 생성해서 고정하였습니다.

```
KUBERNETES_PUBLIC_ADDRESS=$(grep "k8s-controller$" /etc/hosts | awk '{print $1}')


cat << EOF > kubeadm-config.yml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
bootstrapTokens:
- token: "j202s7.tmu57rpu89n8k3f2"
  ttl: "0" 
nodeRegistration:
  criSocket: "/run/containerd/containerd.sock"
localAPIEndpoint:
  advertiseAddress: "${KUBERNETES_PUBLIC_ADDRESS}"
  bindPort: 6443
certificateKey: "94ab545d5869f1d9caf5b7a72211aaf1c893da164342eaaf22ead82ffc1ba6c0"
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: stable
apiServer:
  certSANs:
  - "${KUBERNETES_PUBLIC_ADDRESS}"
  - "192.168.56.11"
  - "192.168.56.12"
  - "192.168.56.13"
  - "10.0.2.15"
controlPlaneEndpoint: "${KUBERNETES_PUBLIC_ADDRESS}:6443"
networking:
  podSubnet: "10.200.0.0/16"
  serviceSubnet: "10.32.0.0/24"
etcd:
  external:
    endpoints:
    - "https://192.168.56.11:2379"
    - "https://192.168.56.12:2379"
    - "https://192.168.56.13:2379"
    caFile: /etc/etcd/ca.crt
    certFile: /etc/etcd/etcd.pem
    keyFile: /etc/etcd/etcd-key.pem
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
EOF
```

Result:

```
kubeadm-config.yml
```

### Kubernetes Image Pull

Kubernetes에서 사용할 이미지들을 미리 가져오겠습니다.

```

kubeadm config images pull

```

### Kubernetes Contoller 구성

작업 대상자 : `k8s-controller-1`
이번 설정은 `k8s-controller-1`에서 만 진행합니다.

```
sudo kubeadm init --upload-certs --config ./kubeadm-config.yaml
```

>output

```
~~~
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 192.168.56.10:6443 --token j202s7.tmu57rpu89n8k3f2 \
	--discovery-token-ca-cert-hash sha256:fed9660155ad16378ebaa0ec09f91f2cb61dea93c19ca837a41e10210b79ec21 \
	--control-plane --certificate-key 94ab545d5869f1d9caf5b7a72211aaf1c893da164342eaaf22ead82ffc1ba6c0

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.56.10:6443 --token j202s7.tmu57rpu89n8k3f2 \
	--discovery-token-ca-cert-hash sha256:fed9660155ad16378ebaa0ec09f91f2cb61dea93c19ca837a41e10210b79ec21

```

### Join Kubernetes Controller

output에서 출력된 정보를 가지고 Controller-2, Controller-3을 kubernetes에 조인하도록 하겠습니다.

```
sudo kubeadm join 192.168.56.10:6443 --token j202s7.tmu57rpu89n8k3f2 \
	--discovery-token-ca-cert-hash sha256:fed9660155ad16378ebaa0ec09f91f2cb61dea93c19ca837a41e10210b79ec21 \
	--control-plane --certificate-key 94ab545d5869f1d9caf5b7a72211aaf1c893da164342eaaf22ead82ffc1ba6c0
```

### Setting kubectl admin.conf

kubectl을 편안하게 사용하게 위해 `k8s-controller-1`에서 생성된 admin.conf를 이용하여 kubeconfig를 구성합니다.

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Verification

```
kubectl get nodes
```

> output

```
NAME               STATUS     ROLES                  AGE   VERSION
k8s-controller-1   NotReady   control-plane,master   24m   v1.22.4
k8s-controller-2   NotReady   control-plane,master   56s   v1.22.4
k8s-controller-3   NotReady   control-plane,master   10s   v1.22.4
```

Next: [Bootstrapping the Kubernetes Worker Nodes](08-bootstrapping-kubernetes-workers.md)
