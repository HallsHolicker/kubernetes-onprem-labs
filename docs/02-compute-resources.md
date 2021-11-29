# Provisioning Compute Resources

Kubernetes는 control plane과 Container가 실행되는 Worker node를 호스팅하기 위한 장비가 필요합니다.
안전하고 가용성이 높은 Kubernetes Cluster를 실행하는데 필요한 컴퓨팅 리소스를 프로비저닝합니다.

## Compute Instances

이미지는 `centos8-stream`으로 별도로 만든 이미지를 사용하였습니다.

Control Plane Node 3개, Worker Node 3개, Router 1개, Client 1개를 생성하겠습니다.

```
Vagrant.configure("2") do |config|
  config.vm.box = "hallsholicker/centos8-stream-k8s"

  config.vm.define "k8s-controller-1" do |controller1|
    controller1.vm.network "private_network", ip: "192.168.56.11"
    controller1.vm.provider "virtualbox" do |v|
      v.customize ["modifyvm", :id, "--groups", "/kubernetes-onprem-labs"]
      v.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
      v.name = "k8s-controller-1"
      v.memory = 2048
      v.cpus = 2
      v.linked_clone = true
    end
    controller1.vm.provision "shell", inline: <<-SHELL
      sudo -i
      sudo echo "192.168.57.0/24 via 192.168.56.100" >> /etc/sysconfig/network-scripts/route-enp0s8
      sudo echo "192.168.58.0/24 via 192.168.56.100" >> /etc/sysconfig/network-scripts/route-enp0s8
      sudo ip route add 192.168.57.0/24 via 192.168.56.100
      sudo ip route add 192.168.58.0/24 via 192.168.56.100
    
    SHELL
  end

  config.vm.define "k8s-controller-2" do |controller2|
    controller2.vm.network "private_network", ip: "192.168.56.12"
    controller2.vm.provider "virtualbox" do |v|
      v.customize ["modifyvm", :id, "--groups", "/kubernetes-onprem-labs"]
      v.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
      v.name = "k8s-controller-2"
      v.memory = 2048
      v.cpus = 2
      v.linked_clone = true
    end
    controller2.vm.provision "shell", inline: <<-SHELL
      sudo -i
      sudo echo "192.168.57.0/24 via 192.168.56.100" >> /etc/sysconfig/network-scripts/route-enp0s8
      sudo echo "192.168.58.0/24 via 192.168.56.100" >> /etc/sysconfig/network-scripts/route-enp0s8
      sudo ip route add 192.168.57.0/24 via 192.168.56.100
      sudo ip route add 192.168.58.0/24 via 192.168.56.100
    
    SHELL
  end

  config.vm.define "k8s-controller-3" do |controller3|
    controller3.vm.network "private_network", ip: "192.168.56.13"
    controller3.vm.provider "virtualbox" do |v|
      v.customize ["modifyvm", :id, "--groups", "/kubernetes-onprem-labs"]
      v.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
      v.name = "k8s-controller-3"
      v.memory = 2048
      v.cpus = 2
      v.linked_clone = true
    end
    controller3.vm.provision "shell", inline: <<-SHELL
      sudo -i
      sudo echo "192.168.57.0/24 via 192.168.56.100" >> /etc/sysconfig/network-scripts/route-enp0s8
      sudo echo "192.168.58.0/24 via 192.168.56.100" >> /etc/sysconfig/network-scripts/route-enp0s8
      sudo ip route add 192.168.57.0/24 via 192.168.56.100
      sudo ip route add 192.168.58.0/24 via 192.168.56.100
      
    SHELL
  end

  config.vm.define "k8s-worker-1" do |worker1|
    worker1.vm.network "private_network", ip: "192.168.56.21"
    worker1.vm.provider "virtualbox" do |v|
      v.customize ["modifyvm", :id, "--groups", "/kubernetes-onprem-labs"]
      v.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
      v.name = "k8s-worker-1"
      v.memory = 1536
      v.cpus = 2
      v.linked_clone = true
    end
    worker1.vm.provision "shell", inline: <<-SHELL
      sudo -i
      sudo echo "192.168.57.0/24 via 192.168.56.100" >> /etc/sysconfig/network-scripts/route-enp0s8
      sudo echo "192.168.58.0/24 via 192.168.56.100" >> /etc/sysconfig/network-scripts/route-enp0s8
      sudo ip route add 192.168.57.0/24 via 192.168.56.100
      sudo ip route add 192.168.58.0/24 via 192.168.56.100
    
    SHELL
  end

  config.vm.define "k8s-worker-2" do |worker2|
    worker2.vm.network "private_network", ip: "192.168.56.22"
    worker2.vm.provider "virtualbox" do |v|
      v.customize ["modifyvm", :id, "--groups", "/kubernetes-onprem-labs"]
      v.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
      v.name = "k8s-worker-2"
      v.memory = 1536
      v.cpus = 2
      v.linked_clone = true
    end
    worker2.vm.provision "shell", inline: <<-SHELL
      sudo -i
      sudo echo "192.168.57.0/24 via 192.168.56.100" >> /etc/sysconfig/network-scripts/route-enp0s8
      sudo echo "192.168.58.0/24 via 192.168.56.100" >> /etc/sysconfig/network-scripts/route-enp0s8
      sudo ip route add 192.168.57.0/24 via 192.168.56.100
      sudo ip route add 192.168.58.0/24 via 192.168.56.100
    
    SHELL
  end

  config.vm.define "k8s-worker-3" do |worker3|
    worker3.vm.network "private_network", ip: "192.168.57.23"
    worker3.vm.provider "virtualbox" do |v|
      v.customize ["modifyvm", :id, "--groups", "/kubernetes-onprem-labs"]
      v.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
      v.name = "k8s-worker-3"
      v.memory = 1536
      v.cpus = 2
      v.linked_clone = true
    end
    worker3.vm.provision "shell", inline: <<-SHELL
      sudo -i
      sudo echo "192.168.56.0/24 via 192.168.56.100" >> /etc/sysconfig/network-scripts/route-enp0s8
      sudo echo "192.168.58.0/24 via 192.168.56.100" >> /etc/sysconfig/network-scripts/route-enp0s8
      sudo ip route add 192.168.56.0/24 via 192.168.57.100
      sudo ip route add 192.168.58.0/24 via 192.168.57.100
    
    SHELL
  end
  
  config.vm.define "k8s-router" do |router|
    router.vm.network "private_network", ip: "192.168.56.100"
    router.vm.network "private_network", ip: "192.168.57.100"
    router.vm.network "private_network", ip: "192.168.58.100"
    router.vm.provider "virtualbox" do |v|
      v.customize ["modifyvm", :id, "--groups", "/kubernetes-onprem-labs"]
      v.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
      v.name = "k8s-router"
      v.memory = 1024
      v.cpus = 1
      v.linked_clone = true
    end
    router.vm.provision "shell", inline: <<-SHELL
      sudo -i
      sudo echo "net.ipv4.ip_forward=1" > /etc/sysctl.conf
      sudo sysctl -p

      SHELL
  end

  config.vm.define "k8s-client" do |client|
    client.vm.network "private_network", ip: "192.168.58.10"
    client.vm.provider "virtualbox" do |v|
      v.customize ["modifyvm", :id, "--groups", "/kubernetes-onprem-labs"]
      v.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
      v.name = "k8s-client"
      v.memory = 1024
      v.cpus = 1
      v.linked_clone = true
    end
    client.vm.provision "shell", inline: <<-SHELL
      sudo -i
      sudo echo "192.168.56.0/24 via 192.168.56.100" >> /etc/sysconfig/network-scripts/route-enp0s8
      sudo echo "192.168.57.0/24 via 192.168.56.100" >> /etc/sysconfig/network-scripts/route-enp0s8
      sudo ip route add 192.168.56.0/24 via 192.168.58.100
      sudo ip route add 192.168.57.0/24 via 192.168.58.100
    SHELL
  end
end





vagrant up
```


## Configuring SSH Access & /etc/hosts


K8S를 연습하기 전에 서버 SSH 접속, 퍠키지 설치 등 기본적인 세팅을 진행 합니다.
K8S 설정 작업은 `k8s-client`에서 진행을 할 예정이며, 원활한 접속을 위해 k8s-client의 ssh key 및 hostname을 전체 서버에 설정하겠습니다.

Ansible 설정을 github에서 가져옵니다.
Ansible은 main.yml은 주석 처리 되어 있으며, 필요한 부분까지 주석을 해제해서 실행하시면 됩니다.
주석 해제 파일 위치 ansible/roles/defaults/tasks/main.yml

```
vagrant ssh k8s-client
sudo dnf -y install ansible git

git clone https://github.com/HallsHolicker/kubernetes-onprem-labs.git
```

SSH fingerprint 정보를 확인해서 known_hosts에 등록합니다.
또한 SSH Key 복사, hostname 설정, /etc/hosts 등록을 합니다. 
최초 설정시에는 ssh key 복사가 되지 않았기에 패스워드를 입력하기 위해 -k 옵션을 설정해서 실행하고, 추후는 -k 옵션 없이도 가능합니다.
password : vagrant
```
cd kubernetes-onprem-labs/ansible
ansible-playbook -i inventories/hosts prerequirement.yml -k
```

이후 챕터의 진행을 자동으로 원할 경우 다음 파일의 주석을 해제하면 됩니다.
```
ansible-playbook -i inventories/hosts main.yml
```


Next: [Installing the Client Tools](03-client-tools.md)
