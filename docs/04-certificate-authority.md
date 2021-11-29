# Provisioning a CA and Generating TLS Certificates

이번 구성은 CloudFlare의 PKI toolkit인 [cfssl](https://github.com/cloudflare/cfssl)를 사용하여 사설 CA 및 ETCD용 인증서를 만들도록 하겠습니다. 만들어지는 파일 및 CA는 인증기관(Certificate Authority) 부트스트랩 및 다음 구성 요소의 TLS 인증에 사용 됩니다.
구성요소 :
* etcd
* kube-apiserver
* kube-controller-manager
* kube-scheduler
* kubelet
* kube-proxy

## Certificate Authority

이번 섹션에서는 추가적인 TLS 인증서를 생성하는데 필요한 인증기관(Certificate Authority)을 구성합니다.
CA 설정 파일, 인증서, 개인키를 생성합니다.
```
{
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
}
```

Results:

```
ca-key.pem
ca.pem
```

### Kubernetes Private SA를 위햬 pem 파일을 crt로 변경

```
openssl x509 -inform pem -in ca.pem -out ca.crt
openssl rsa -inform pem -in ca-key.pem -out ca.key

```

Results:
```
ca.crt
ca.key
```


### The ETCD Server Certificate

원격 클라이언트의 유효성을 인증하기 위해 Kubernetes API 서버 인증서에 Client IP, kubernetes API VIP 정보를 포함합니다.

Kubernetes API 서버 인증서와 개인키 생성:

```
{
KUBERNETES_PUBLIC_ADDRESS=$(grep "k8s-controller$" /etc/hosts | awk '{print $1}')
KUBERNETES_CLIENT_ADDRESS=$(grep "k8s-client" /etc/hosts | awk '{print $1}')
KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

cat > etcd-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes onprem labs",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.crt \
  -ca-key=ca.key \
  -config=ca-config.json \
  -hostname=10.32.0.1,192.168.56.10,192.168.56.11,192.168.56.12,192.168.56.13,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  etcd-csr.json | cfssljson -bare etcd
}
```

Results:

```
etcd-key.pem
etcd.pem
```

## Distribute the Client and Server Certificates

각 Worker node에 certificate와 Private key를 복사합니다:

```
for hostname in k8s-worker-1 k8s-worker-2 k8s-worker-3; do
  scp ca.crt ${hostname}:~/
done
```

각 Controller node에 certificate와 Private key를 복사합니다:

```
for hostname in k8s-controller-1 k8s-controller-2 k8s-controller-3; do
  scp ca.crt ca.key etcd-key.pem etcd.pem ${hostname}:~/
done
```

Next: [Bootstrapping the etcd Cluster](05-bootstrapping-etcd.md)
