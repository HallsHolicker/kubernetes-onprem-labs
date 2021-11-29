# Provisioning Pod Network Routes

External Loadbalancer를 구성하기 위하여 metalLB를 설치하도록 하겠습니다.

## Install MetalLB

```
ssh k8s-client
```

```
curl https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/namespace.yaml -o metallb_namespace.yaml
curl https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/metallb.yaml -o metallb.yaml
```

```
kubectl apply -f metallb_namespace.yaml
```

```
kubectl apply -f metallb.yaml
```

다음 명령어는 최초 설치시에만 실행합니다.

```
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

MetalLB ConfigMap 생성:

```
cat << EOF | kubectl create -f -
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    peers:
    - peer-address: 192.168.56.100
      peer-asn: 64501
      my-asn: 64500
    address-pools:
    - name: default
      protocol: bgp
      addresses:
      - 192.168.72.0/24
EOF
```

## Verification MetalLB

MetalLB 설치가 잘 동작 되는 지 확인 합니다.

```
kubectl get pods -n metallb-system
```

> output

```
NAME                          READY   STATUS    RESTARTS     AGE
controller-7dcc8764f4-f4twh   1/1     Running   1 (5s ago)   35s
speaker-2fxqp                 1/1     Running   0            35s
speaker-9n6kt                 1/1     Running   0            35s
speaker-bgbpj                 1/1     Running   0            35s
speaker-pw96p                 1/1     Running   0            35s
speaker-qclgc                 1/1     Running   0            35s
speaker-t55gg                 1/1     Running   0            35s
```



## FRRouting 설치

MetalLB를 BGP모드로 동작시키고 있으며, BGP로 전파가 제대로 되는지 확인 하기 위해 FRRouting을 `k8s-client`에 설치해서 BGP를 연결해 봅니다.

```
FRRVER="frr-stable"
curl -O https://rpm.frrouting.org/repo/$FRRVER-repo-1-0.el8.noarch.rpm
sudo dnf -y install ./$FRRVER*
sudo dnf -y install frr frr-pythontools
```

BGP Daemon 사용
```
sudo sed -i 's/^bgpd=no/bgpd=yes/' /etc/frr/daemons
sudo sed -i 's/^vtysh_enable=no/vtysh_enable=yes/' /etc/frr/daemons
sudo sed -i 's/^#MAX_FDS=1024/MAX_FDS=1024/' /etc/frr/daemons
```

BGP 설정
```
cat << EOF | sudo tee /etc/frr/frr.conf
frr version 8.0
frr defaults traditional
hostname localhost.localdomain
log syslog informational
no ipv6 forwarding
!
router bgp 64501
 no bgp ebgp-requires-policy
 neighbor k8s peer-group
 neighbor k8s remote-as 64500
 bgp listen range 192.168.56.0/22 peer-group k8s
 !
 address-family ipv4 unicast
  neighbor k8s soft-reconfiguration inbound
 exit-address-family
!
line vty
!
EOF
```

FRR 재시작
```
sudo systemctl restart frr
```
