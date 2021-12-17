# Provisioning Pod Network Routes

이번 실습에서는 Kubernetes CNI로 calico를 설치하도록 하겠습니다. 또한. External Loadbalancer를 구성하기 위해여 metalLB도 설치하도록 하겠습니다.

## Installing kubernetes CNI calico

이번 작업은 `k8s-client`에서 진행합니다. 

```
ssh k8s-client
```

Download Calico manifests

```
curl https://docs.projectcalico.org/manifests/calico.yaml -o calico.yaml
```

Calico Version check

```
 less calico.yaml | grep image
```

> output

```
          image: docker.io/calico/cni:v3.21.1
          image: docker.io/calico/cni:v3.21.1
          image: docker.io/calico/pod2daemon-flexvol:v3.21.1
          image: docker.io/calico/node:v3.21.1
          image: docker.io/calico/kube-controllers:v3.21.1
```

```
kubectl apply -f calico.yaml
```

## Verification Calico

Calico가 정상 동작하기까지는 약 2~3분 정도 소요됩니다.
무선 환경에서 배포할 경우 인터넷 환경에 따라 1시간 이상이 소요될 수도 있습니다. -0-

```
kubectl get pods -n kube-system
```

> output

```
NAME                                       READY   STATUS    RESTARTS      AGE
calico-kube-controllers-56b8f699d9-w9p25   1/1     Running   0             3m9s
calico-node-9bdvv                          1/1     Running   0             3m9s
calico-node-c7hrj                          1/1     Running   1 (98s ago)   3m9s
calico-node-jb7d8                          1/1     Running   1 (85s ago)   3m9s
calico-node-ld6ht                          1/1     Running   0             3m9s
calico-node-rct7m                          1/1     Running   0             3m9s
calico-node-zj5qr                          1/1     Running   0             3m9s
```


Next: [Provisioning Loadbalancer Network Metallb](10-provisioning-lb-network-metallb.md)
