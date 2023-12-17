# install node_exporter

- download and extract node exporter
download binary node_exporter, untuk realses latest bisa di lihat [disini](https://github.com/prometheus/node_exporter/releases/)
```
mkdir ~/node-exporter && cd ~/node-exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
```

create node_exporter.servvice
```
cat << 'EOF' > node_exporter.service
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=root
Group=root
Type=simple
ExecStart=/usr/local/bin/node_exporter \
    --log.level=error \
    --no-collector.btrfs \
    --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|run/k3s/containerd/.+|run/docker/.+|var/lib/docker/.+|var/lib/kubelet/pods/.+)($|/) \
    --collector.filesystem.fs-types-exclude=^(autofs|binfmt_misc|bpf|cgroup2?|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|iso9660|mqueue|nsfs|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|selinuxfs|squashfs|sysfs|tmpfs|tracefs)$ \
    --collector.netclass.ignored-devices=^(cali.*|docker.*|flannel.*|lo|nodelocaldns|tap.*|veth.*|vnet.*|ens[0-4]|virbr[0-9]+-nic|[a-f0-9]{15})$ \
    --collector.netdev.device-exclude=^(cali.*|docker.*|flannel.*|lo|nodelocaldns|tap.*|veth.*|vnet.*|virbr[0-9]+-nic|[a-f0-9]{15})$ \

[Install]
WantedBy=multi-user.target
EOF
```

- create script for install node_exporter
```
cat << 'EOF' > install_exporter.sh
tar -xvzf node_exporter-1.7.0.linux-amd64.tar.gz && rm node_exporter-1.7.0.linux-amd64.tar.gz
mv node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/
sudo mv node_exporter.service /etc/systemd/system/node_exporter.service
sudo systemctl daemon-reload
sudo systemctl restart node_exporter
sudo systemctl status node_exporter
EOF
```

```
chmod +x install_exporter.sh
```

- install all node
```
for i in {0..4}; do \
ssh -i ~/access.pem root@10.20.10.1$i hostname; \
scp -i ~/access.pem -r ./* root@$10.20.10.1$i:~/ ;\
ssh -i ~/access.pem root@10.20.10.1$i bash install_exporter.sh && rm install_exporter.sh;\
done
```

- verifikasi
```
for i in {0..4}; do \
ssh -i ~/access.pem.pem root@$10.20.10.1$i 'hostname; node_exporter --version'; \
done
```
