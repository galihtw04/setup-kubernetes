# Create Volume NFS Provisioner
- case disini saya akan membuat nfs server yang nanti digunakan oleh cluster untuk menjadi penyimpanan volume

Setup nfs server1 in master-01
 Disini kita akan menentukan target untuk menjadi penyimmpanan nfs server. sebelumnya saya sudah set lvm untuk tutorialnya ada disini [create lvm](https://github.com/galihtw04/Libvirt-Terraform/blob/main/Lab/disk/vg_lvm.md#create-vg)

```
lsblk
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/93dc9a2a-d241-4f29-b7ca-7c0dc5ed516d)

install nfs server
```
chown -R nobody:nogroup /nfs-share
chmod 777 -R /nfs-share
apt install nfs-kernel-server -y
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/dfd8ccf8-8162-4c1b-a2f4-c899e0d5074d)

sharing dir sharing /nfs-share
```
echo "/nfs-share *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/7634cedf-840d-4841-8b46-8b068699850c)

restart and check nfs-server
```
systemctl restart nfs-kernel-server

showmount -e
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/dc16c9f6-bff3-4433-884d-90a126c99c4b)

install nfs client in all node
```
for x in {0..4}; do ssh -i access.pem 10.20.10.1$x 'hostname; apt install nfs-common -y' ;done
```

install helm all node
```
cat << 'EOF' >> install_helm.sh
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
EOF

chmod +x install_helm.sh
```

```
for x in {0..4}; do scp -i access.pem install_helm.sh root@10.20.10.1$x:~/ ; \
ssh -i access.pem 10.20.10.1$x 'hostname; bash ~/install_helm.sh' ;done
```

deploy nfs provisioner external
```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

helm repo list
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/10cc2042-9e75-459b-ae61-f28b1a1209ac)

```
helm install -n kube-system nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
--set nfs.server=10.20.10.10 --set nfs.path=/nfs-share
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/cefe3d9e-26fd-46cb-92b7-813db3dd6771)

checking pods nfs
```
kubectl get -n kube-system pods -o wide| grep nfs
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/e0a2b6cc-8bf7-4c10-b8ff-a34ac61927b7)

Set as default storage class
```
kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

testing
flow nanti kita akan membuat persisten volume yang nanti akan digunakan oleh deployment nginx, nanti kita akan membuat service juga untuk expose nginx, dan juga membuat Ingress untuk deployment nginx.

```
cat << 'EOF' > testing-vol.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: vol-testing
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: storage-testing
  namespace: vol-testing
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-testing-vol
  namespace: vol-testing
spec:
  selector:
    matchLabels:
      app: app-testing-vol
  replicas: 3
  template:
    metadata:
      labels:
        app: app-testing-vol
    spec:
      volumes:
      - name: storage-testing
        persistentVolumeClaim:
          claimName: storage-testing
      containers:
      - name: app-testing-vol
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: storage-testing
          mountPath: /usr/share/nginx/html
---
apiVersion: v1
kind: Service
metadata:
  name: service-vol
  namespace: vol-testing
spec:
  type: ClusterIP
  selector:
    app: app-testing-vol
  ports:
  - port: 80
    name: http
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: vol-testing
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: nginx-vol.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service-vol
            port:
              number: 80
EOF
```

apply manifest testing-vol.yaml
```
kubectl apply -f testing-vol.yaml
```

checking
```
kubectl get all -n vol-testing -o wide
kubectl get -n vol-testing pvc
kubectl get -n vol-testing ingress
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/fdf51bc6-a27f-465b-a4a9-9e6d6cc42fd1)
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/dc4d5137-975c-41dc-ae0c-79e81724adb2)
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/ef33d89e-065c-4d72-a365-f8cb824ae87a)

test curl
```
curl -H 'Host: nginx-vol.local' 10.20.10.151 -I
```
> disni forbiden karena tidak ada konten pada nginxnyua
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/c69e99dc-51e2-45b8-8c24-d345db05468d)

create konten
```
echo 'Hello Word' > /nfs-share/vol-testing-storage-testing-pvc-b54976d1-7888-4641-98d6-4fa1fbb6ad89/index.html
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/9dbf5b2a-36aa-4afc-babe-fe4025b15966)
> direcotry yang kita tuju adalah yang bukan archived, archived adalah data dari deployment yang sudah tidak ada atau yang sudah terhapus dan kita membuat lagi deployment maka akan terbuat lagi dan yang sebelumnya tidak akan di hapus dan buat archived.

![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/1d4a7b9d-37e5-43d1-a056-3b650295a5e9)

curl again.
```
curl -H 'Host: nginx-vol.local' 10.20.10.151 -I
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/5529694e-4161-4bf6-90fd-33040bbaefc8)
