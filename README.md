# setup-kubernetes

- Enviroment

| Hostname | Network | vCPU | RAM | Storage | OS |
| ------ | ------ | ------ | ------ | ------ | ------ |
| k8s-student-master01 | 10.20.10.10 | 4 | 8Gib | 20G | Ubuntu22.04 |
| k8s-student-master02 | 10.20.10.11 | 4 | 8Gib | 20G | Ubuntu22.04 |
| k8s-student-master03 | 10.20.10.12 | 4 | 8Gib | 20G | Ubuntu22.04 |
| k8s-student-worker01 | 10.20.10.13 | 4 | 8Gib | 50G | Ubuntu22.04 |
| k8s-student-worker02 | 10.20.10.14 | 4 | 8Gib | 50G | Ubuntu22.04 |

add emviroment to /etc/hosts
```
cat << 'EOF' >> /etc/hosts
#k8s api
10.20.10.100 k8s.student.cloud
#node master
10.20.10.10 k8s-student-master01 
10.20.10.11 k8s-student-master02
10.20.10.12 k8s-student-master03
#node worker
10.20.10.13 k8s-student-worker01
10.20.10.14 k8s-student-worker02
EOF
```

output

![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/3f076a10-617b-4a76-805b-4a6b76d850e6)

copy file /etc/hosts to master2,master3 and all workers
```
for x in {1..4}; do ssh -i access.pem 10.20.10.1$x hostname; \
scp -i access.pem /etc/hosts root@10.20.10.1$x:/etc/ ;done
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/db24e4a7-0840-4fc1-bb05-503a14f92df2)

verify
```
for x in {0..4}; do ssh -i access.pem 10.20.10.1$x hostname; \
ssh -i access.pem 10.20.10.1$x cat /etc/hosts | grep 10.20.10. ;done
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/e5123da9-70fb-4750-a990-f8a721422894)


install package
```
for x in {0..4}; do ssh -i access.pem 10.20.10.1$x apt-get update \
&& apt-get upgrade -y --with-new-pkgs;
ssh -i access.pem 10.20.10.1$x apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release ;done
```

verify
```
for i in apt-transport-https ca-certificates curl gnupg lsb-release; do
    for x in {0..4}; do
        ssh -i access.pem 10.20.10.1$x "hostname && dpkg -l | grep $i"
    done
done
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/6f298520-50b3-4301-8966-558ecd0c0bab)

disable swap
> kita akan disable swap karena ada beberapa pertimbangan disini kita akan menggunakan cgroup dan kita mendisable swap agar pod/app kita bisa berjalan dengan stabil
```
for x in {0..4}; do ssh -i access.pem 10.20.10.1$x swapoff -a; \
ssh -i access.pem 10.20.10.1$x "sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab" ;done
```

verify 
```
for x in {0..4}; do ssh -i access.pem 10.20.10.1$x 'hostname; free -h && swapon --show' ;done
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/07c1db4e-0881-4877-86fc-aea6718a7265)

- setup cri containerd
create script cri containerd
```
cat << 'EOF' >> cri_containerd.sh
#!/bin/bash
# Load containerd required modules
cat <<TOF | tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
TOF
modprobe overlay
modprobe br_netfilter
# Setup required sysctl params, these persist across reboots.
cat <<END | tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
END

# set ip forward
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sysctl -p

# Apply sysctl params without reboot
sysctl --system
#repository
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs)
stable" && apt update
# Install containerd
apt-get install -y containerd
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl start containerd.service
systemctl restart containerd.service
EOF
```

Add permission execute
```
chmod +x cri_containerd.sh
```

install containerd all vm/instance master and worker
```
for x in {0..4}; do scp -i access.pem cri_containerd.sh root@10.20.10.1$x:~/ ; \
ssh -i access.pem 10.20.10.1$x bash ~/cri_containerd.sh ;done
```

verify service containerd
```
for x in {0..4}; do ssh -i access.pem 10.20.10.1$x 'hostname; systemctl status containerd.service | grep Active' ;done
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/7e7080d3-4a2e-452e-9fb0-003657e3c982)

# install kubernetes
create script
```
cat << 'EOF' >> install_kubernetes.sh
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo 'deb http://apt.kubernetes.io/ kubernetes-xenial main' > /etc/apt/sources.list.d/kubernetes.list
apt-get update && apt-get install -y kubelet=1.25.0-00 kubeadm=1.25.0-00 kubectl=1.25.0-00
apt-mark hold kubelet kubeadm kubectl
EOF
```

install all node
```
chmod +x install_kubernetes.sh
for x in {0..4}; do scp -i access.pem install_kubernetes.sh root@10.20.10.1$x:~/ ; \
ssh -i access.pem 10.20.10.1$x bash ~/install_kubernetes.sh ;done
```

verify
```
for x in {0..4}; do ssh -i access.pem 10.20.10.1$x "hostname; kubelet --version; kubectl version -o yaml | grep gitVersion"; \
ssh -i access.pem 10.20.10.1$x kubeadm version | awk -F: '{print $5}'; done
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/d6f636e3-e5b8-44fd-9cfa-b3359ab995f2)

autocomplate command kubernetes
> by default setelah kita install kubernetes kita tidah bisa menggunakan *Tab* untuk autocomplate
```
cat << 'EOF' >> autocomplate-k8s.sh
echo "source <(kubectl completion bash)" >> ~/.bashrc
echo "alias k=kubectl" >> ~/.bashrc
source ~/.bashrc
EOF

chmod +x autocomplate-k8s.sh
for x in {0..2}; do scp -i access.pem autocomplate-k8s.sh root@10.20.10.1$x:~/ ; \
ssh -i access.pem 10.20.10.1$x bash ~/autocomplate-k8s.sh ;done
```


# Setup HA Master
install package haproxy and keepalived(vrrp)
> destinationnya cuma master01,master02,dan master03
```
cat << 'EOF' >> vrrp-install.sh
apt-get install haproxy keepalived -y
groupadd -r keepalived_script
useradd -r -s /sbin/nologin -g keepalived_script -M keepalived_script
EOF

chmod +x vrrp-install.sh
for x in {0..2}; do scp -i access.pem vrrp-install.sh root@10.20.10.1$x:~/ ; \
ssh -i access.pem 10.20.10.1$x bash ~/vrrp-install.sh ;done
```

```
cat << 'EOF' >> /etc/keepalived/check_apiserver.sh
#!/bin/sh
errorExit() {
echo "*** $*" 1>&2
exit 1
}
curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error GET https://localhost:6443/"
if ip addr | grep -q 10.20.10.100; then
    curl --silent --max-time 2 --insecure https://k8s.student.cloud:8443/ -o /dev/null || errorExit "Error GET https://k8s.student.cloud:8443"
fi
EOF

chmod 755 /etc/keepalived/check_apiserver.sh
for x in {1..2}; do scp -i access.pem /etc/keepalived/check_apiserver.sh root@10.20.10.1$x:/etc/keepalived/check_apiserver.sh ;done
```


setup vrrp on k8s-student-master01
```
cat << 'EOF' >> /etc/keepalived/keepalived.conf
global_defs {
    router_id LVS_DEVEL
    enable_script_security
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance k8s-apiserver-hokage {
    state MASTER
    interface ens3
    virtual_router_id 51
    priority 100
    authentication {
        auth_type PASS
        auth_pass s3Cur3
    }
    virtual_ipaddress {
        10.20.10.100/24
    }
    track_script {
        check_apiserver
    }
}
EOF
```

![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/80eaa28b-f1e5-498a-bf8f-29374ea90f32)

setup keepalived on k8s-student-master02 backup
```
cat << 'EOF' >> /etc/keepalived/keepalived.conf
global_defs {
    router_id LVS_DEVEL
    enable_script_security
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance k8s-apiserver-hokage {
    state BACKUP
    interface ens3
    virtual_router_id 51
    priority 99
    authentication {
        auth_type PASS
        auth_pass s3Cur3
    }
    virtual_ipaddress {
        10.20.10.100/24
    }
    track_script {
        check_apiserver
    }
}
EOF
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/34e6a11f-31b3-400a-a66b-e8273a0a76ca)


setup keepalived on k8s-student-master03 backup
```
cat << 'EOF' >> /etc/keepalived/keepalived.conf
global_defs {
    router_id LVS_DEVEL
    enable_script_security
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance k8s-apiserver-hokage {
    state BACKUP
    interface ens3
    virtual_router_id 51
    priority 98
    authentication {
        auth_type PASS
        auth_pass s3Cur3
    }
    virtual_ipaddress {
        10.20.10.100/24
    }
    track_script {
        check_apiserver
    }
EOF
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/8486a6c7-d7e5-445f-ac39-0b99299278ca)

enable and start keepalived
```
for x in {0..2}; do ssh -i access.pem 10.20.10.1$x 'hostname; systemctl enable keepalived; systemctl start keepalived' ;done
```

verify
```
for x in {0..2}; do ssh -i access.pem 10.20.10.1$x 'hostname; systemctl status keepalived | grep Active' ;done
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/ed578742-20ab-4ceb-812b-1df855b3a295)


# configuration haproxy
> # Note
> execute on k8s-student-master01 k8s-student-master02
```
cat << 'EOF' >> /etc/haproxy/haproxy.cfg
frontend apiserver
        bind *:8443
        mode tcp
        option tcplog
        default_backend apiserver

backend apiserver
        option httpchk GET /healthz
        http-check expect status 200
        mode tcp
        option ssl-hello-chk
        balance roundrobin
        server master01 k8s-student-master01:6443 check fall 3 rise 2
        server master02 k8s-student-master02:6443 check fall 3 rise 2
        server master03 k8s-student-master03:6443 check fall 3 rise 2
EOF

for x in {1..2}; do scp -i access.pem /etc/haproxy/haproxy.cfg root@10.20.10.1$x:/etc/haproxy/haproxy.cfg ;done
```

enable and start haproxy
```
for x in {0..2}; do ssh -i access.pem 10.20.10.1$x 'hostname; systemctl enable haproxy; systemctl start haproxy' ;done
```

verify running haproxy
```
for x in {0..2}; do ssh -i access.pem 10.20.10.1$x 'hostname; systemctl status haproxy | grep Active' ;done
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/ba63a63d-5d49-438a-9538-d9d88ae69a7d)

# setup cluster kubernetes (execute in master01)
- Konfigurasi Boostrap First Control panel, execute in master01
```
cat << 'EOF' > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: "v1.25.0"
controlPlaneEndpoint: "k8s.student.cloud:6443"
clusterName: k8s-student
networking:
  podSubnet: 172.16.100.0/16
  serviceSubnet: 192.168.20.0/12
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/f1ac6872-f995-4952-983a-f1d5abe60bc6)

- kubeadm init
```
kubeadm init --config kubeadm-config.yaml --upload-certs
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/3c634eae-b036-4fc8-a305-3ed05e5d4b41)

create direcotry kube
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

edit kubeadm
```
kubectl -n kube-system get configmap kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' > kubeadm.yaml
sed -i 's/6443/8443/g' kubeadm.yaml
kubeadm init phase upload-config kubeadm --config kubeadm.yaml
```

edit port kubeproxy
```
kubectl -n kube-system get cm kube-proxy -o yaml > kube-proxy.yaml
sed -i 's/6443/8443/g' kube-proxy.yaml
kubectl apply -f kube-proxy.yaml
```

restart kubeadm dan kubeproxy
```
kubectl -n kube-system rollout restart ds kube-proxy
kubectl -n kube-system rollout status ds kube-proxy
```

update kube-info
```
kubectl -n kube-public get cm cluster-info -o yaml > cluster-info.yaml
sed -i 's/6443/8443/g' cluster-info.yaml
kubectl apply -f cluster-info.yaml
```

update config admin.conf and kubelete.conf
```
sed -i 's/6443/8443/g' .kube/config /etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf
systemctl restart kubelet
```

verify
```
kubectl cluster-info

grep server /etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/0739ebb2-dd5c-4424-b682-30a92e3f359d)

# join Control-plane and worker
- create token
```
echo "$(kubeadm token create --print-join-command) --control-plane --certificate-key $(kubeadm init phase upload-certs --upload-certs | grep -vw -e certificate -e Namespace)"
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/ca571cc0-619d-43b7-826b-67038661edbf)


join master2 dan master3
> # note
> copy ouput pada command sebelumnya, yang akan kita gunakan untuk join ke cluster dengan role control-plane
```
cat << 'EOF' >> join_master.sh
kubeadm join k8s.student.cloud:8443 --token o8htj5.8zbdkw3meleuei19 --discovery-token-ca-cert-hash sha256:7164cc629f7bdf398406fe18bef0489d5e532123eee9b787032c369f812bb894  --control-plane --certificate-key 83bcb83ae7416e9f7a141250098d145648db07d700754d138747d41f3b6555ee
EOF

chmod +x join_master.sh
for x in {1..2}; do scp -i access.pem join_master.sh root@10.20.10.1$x:~/join_master.sh; ssh -i access.pem 10.20.10.1$x bash ~/join_master.sh ;done
```

verify
> # note
> belum ready karena masih belum di apply cni
```
k get node -o wide
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/576f1f6e-1074-40d6-976d-3d2123cdedde)

create dir kube
```
cat<< 'EOF' >> create-dir.sh
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
EOF

chmod +x create-dir.sh
for x in {1..2}; do scp -i access.pem create-dir.sh root@10.20.10.1$x:~/create-dir.sh; ssh -i access.pem 10.20.10.1$x bash ~/create-dir.sh ;done
```

- join worker
> hapus command --control-plane dan --certificate-key, untuk commandnya akan seperti berikut
```
cat << 'EOF' >> join_worker.sh
kubeadm join k8s.student.cloud:8443 --token o8htj5.8zbdkw3meleuei19 --discovery-token-ca-cert-hash sha256:7164cc629f7bdf398406fe18bef0489d5e532123eee9b787032c369f812bb894
EOF

chmod +x join_worker.sh
for x in {3,4}; do scp -i access.pem join_worker.sh root@10.20.10.1$x:~/join_worker.sh; ssh -i access.pem 10.20.10.1$x bash ~/join_worker.sh ;done
```

verify 
```
kubectl get nodes -o wide
kubectl get all -A
```
> ketika kita show node maka masih nodeready, karena kita belum apply cni(Container network interface) yang akan digunakan pada pod/container

![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/3150cfd0-80df-4656-82ce-06762ac66eb1)

![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/d005b7d8-364f-4542-81b6-aecc48c711a8)

- apply calico
> selain calico masih ada flanel.wave,cillium, dan lain-lain, dari cni-cni tersebut memiliki kelebihan dan kekurangan masing-masing bisa kalian sesuaing dengan eviroment applikasi kalian, cni bisa kita ubah dari calico ke flanel atau yang lain, tergantung dengan reproduce kita.
```
wget https://raw.githubusercontent.com/projectcalico/calico/master/manifests/calico.yaml
kubectl apply -f calico.yaml
```

verify again
```
kubectl get nodes -o wide
kubectl get pods -n kube-system
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/3f9d65e2-6f63-47ab-8269-6795761f7838)

![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/1635ea0f-44d9-4767-a577-ce61b90c7d39)
