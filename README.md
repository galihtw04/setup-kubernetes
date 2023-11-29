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

install package
```
apt-get update && apt-get upgrade -y --with-new-pkgs
apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
```

disable swap
> # kita akan disable swap karena ada beberapa pertimbangan disini kita akan menggunakan cgroup dan kita mendisable swap agar pod/app kita bisa berjalan dengan stabil
```
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
- setup cri containerd
create script cri containerd
```
cat << 'EOF' >> cri_containerd.sh
#!/bin/bash
# Load containerd required modules
cat <<EOF | tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter
# Setup required sysctl params, these persist across reboots. cat <<EOF | tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
# Apply sysctl params without reboot
sysctl --system
#repository
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o
/etc/apt/trusted.gpg.d/docker.gpg
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs)
stable" && apt update
# Install containerd
apt-get install -y containerd
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd
EOF
```

running  cri containerd
```
chmod +x cri_containerd.sh
bash cri_containerd.sh
```

verify service containerd
```
systemcl statu conainerd.service
```

# install kubernetes
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - \
echo 'deb http://apt.kubernetes.io/ kubernetes-xenial main' > \
/etc/apt/sources.list.d/kubernetes.list
apt-get update && apt-get install -y kubelet=1.25.0-00 kubeadm=1.25.0-00 kubectl=1.25.0-00
apt-mark hold kubelet kubeadm kubectl
```

autocomplate command kubernetes
> by default setelah kita install kubernetes kita tidah bisa menggunakan *Tab* untuk autocomplate
```
# set up autocomplete in bash into the current shell, bash-completion package should be installed first. source <(kubectl completion bash)
# add autocomplete permanently to your bash shell. echo "source <(kubectl completion bash)" >> ~/.bashrc
#enable auto complate
. .bashrc
```

# Setup HA Master
install package haproxy and keepalived(vrrp)
> execure on @k8s-student-master01 and k8s-student-master02
```
apt-get install haproxy keepalived -y
```

## konfigure keepalived (vrrp)
- create group and user for executing health keepalived
```
groupadd -r keepalived_script
useradd -r -s /sbin/nologin -g keepalived_script -M keepalived_script
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
```

add permission execute and scp to master-02
```
chmod +x /etc/keepalived/check_apiserver.sh
scp /etc/keepalived/check_apiserver.sh root@k8s-student-master02://etc/keepalived/check_apiserver.sh
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

enable and start keepalived
```
systemctl enable keepalived
systemctl start keepalived
```

# configuration haproxy
> # Note
> execute on k8s-student-master01 and k8s-student-master02
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
        server master02 k8s-student-master01:6443 check fall 3 rise 2
EOF
```

enable and start haproxy
```
systemctl enable haproxy
systemctl start haproxy
```
