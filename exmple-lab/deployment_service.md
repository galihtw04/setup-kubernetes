# create deployment
- create deployment using command
```
kubectl create deployment nginx-deployment --image=nginx:latest
```
confirm
```
kubectl get deployments
kubectl get deployments -o wide
kubectl get pods
kubectl get pods -o wide
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/14a30bae-4779-4a51-82dd-fe606702dc94)
> # Note
> ketika menggunakan comamnd *kubectl get pods -o wide* kita bisa mengetahui pods ditempatkan di woker mana.
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/577e3766-b3dc-400f-a224-5143830541f4)

test access
```
# curl <ip pods tadi>
curl 172.16.122.193
```
> ip pods di dapat dari konfigurasi boostrap kubeadm [link](https://github.com/galihtw04/setup-kubernetes/#setup-cluster-kubernetes-execute-in-master01) pada bagian podSubnet

![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/d4503144-e817-4f36-a738-2bd44bd65566)

- create manifest deployment
1. create template deployemt
```
kubectl create deployment nginx-template --image=nginx:latest -o yaml > nginx_template.yaml
cat nginx_template.yaml
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/2930930a-26fa-4409-a5e2-700b7cbd9f6f)
> # note
> jika kita menggunakan command nginx_tempalte diatas maka akan terbuat deployment nginx_template, jadi deployment nginx-template bisa kalian hapus jika ingin membuat template manifest
> dan untuk manifest harus menggunakn yaml atau yml

confirm deployment nginx-template
```
kubectl get deployment
kubectl get pods
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/e66e5d74-4f7a-4143-99be-a13458d044df)

delete deployment nginx-template
```
kubectl delete deployment nginx-template
kubectl get deployment
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/aca13f33-3306-4634-b564-db7f27fd90b0)

untuk contoh file yang akan kita dapat seperti berikut,
<details><summary>nginx_template.yaml</summary>

```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: "2023-12-05T05:34:03Z"
  generation: 1
  labels:
    app: nginx-template
  name: nginx-template
  namespace: default
  resourceVersion: "264764"
  uid: b117715f-7d7e-4ba4-8ef7-442a5ae48dca
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx-template
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx-template
    spec:
      containers:
      - image: nginx:latest
        imagePullPolicy: Always
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status: {}
```
</details>
disini kita akan coba membuat replica 10, untuk membuet replica 10 kalian edit pada bagian replica yang tadinya 1 menjadi 10.

```
spec:
  progressDeadlineSeconds: 600
  replicas: 10
```

apply manifest
```
k apply -f nginx_template.yaml
```

apply deployment
```
k get deployments
k get pods -o wide | grep nginx-template
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/18adbf1e-4f7e-4ba4-a41f-3b86209fa7d2)

verify
```
kubectl get pods -o wide | grep nginx-template | awk '{print $6}' > nginx-ip.txt

for x in $(cat nginx-ip.txt); do
    if curl -s --head $x | grep -q "200 OK"; then
        echo "Success: $x returned HTTP 200"
    else
        echo "Error: $x did not return HTTP 200"
    fi
done
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/34a7fb7b-c999-4b95-ba11-fb1eaae6667d)
> jik perhatikan kita dapat melakukan hit dengan normal dengan code 200 jika berhasil, kelebihan kita membuat pod menggunakan deployment kita bisa membuat beberapa replica, dan jika ada pods yang ke delete maka pods akan membuat baru sesuai ketentuan replica pada deployments

# Service
Di Kubernetes, Service (atau svc) adalah salah satu objek yang digunakan untuk mengekspos aplikasi atau layanan ke dalam jaringan cluster atau luar cluster. Service membantu dalam mengelola konektivitas antara aplikasi yang berjalan di dalam cluster, memungkinkan komunikasi yang dapat diandalkan antara komponen-komponen tersebut. Berikut adalah beberapa tipe Service yang umum digunakan di Kubernetes.

ClusterIP:

Jenis ini membuat serivce hanya dapat diakses dari dalam cluster.Ketika Anda ingin service hanya dapat diakses oleh komponen internal dalam cluster.
- expose deployment nginx-template
```
kubectl expose deployment nginx-template --type=ClusterIP --name=svc-nginx-template --port=80 --target-port=80
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/2e4ccb30-5cc4-4644-b71f-ecebfc91cc3f)

- testing dalam cluster
```
# curl http://<ip service> -I
curl http://192.162.199.249 -I
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/a1554db0-b0ef-4392-a407-741f7cb6f5b2)

![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/eb493944-ff8b-413b-b54d-ff1ebd346027)

- testing luar cluster
> testing di host libvirt
```
curl http://192.162.199.249 -I -vvv
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/8d4c2352-f0a4-41f9-8be9-a666edd561fe)


NodePort:

Mengekspos layanan melalui port tertentu pada semua node dalam cluster.Untuk mengakses layanan dari luar cluster atau melalui IP node dan port tertentu.

- expose deployment nginx-template
```
kubectl expose deployment nginx-template --type=NodePort --name=svc-nodeport-nginx --port=80 --target-port=80
```
> # note
> ketika kita membuat atau expose ke nodeport maka port yang akan digunakan untuk hit service akan acak, untuk range port pada nodeport adalah 30000-32767.
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/33cfc62b-dc1d-49ca-8162-93f74801eec8)

- testing dalam cluster
```
# curl http://<ip node cluster>:<port nodeport>
curl http://10.20.10.100:30103 -I
curl http://10.20.10.13:30103 -I
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/83c6e9ef-55d4-4d16-9475-dca0d413ae0b)

- testing luar cluster
```
curl http://10.20.10.100:30103 -I
curl http://10.20.10.13:30103 -I
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/04eb9183-f860-4d3b-82a1-9a830adc50fa)

LoadBalancer:

Mengekspos service melalui load balancer eksternal (dapat menjadi layanan cloud atau perangkat keras fisik).Ketika Anda ingin mendistribusikan lalu lintas service secara eksternal melalui load balancer.
- expose deployment nginx-template
```
kubectl expose deployment nginx-template --type=LoadBalancer --name=svc-lb-nginx --port=80 --target-port=80
kubectl get svc
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/ef4c7b76-092a-451b-814d-6c65850b75e3)
> Note
> 
> jika kalian perhatikan pada bagian external ip terdapat yang pending, external ip digunakan untuk diakses menggunakan port asli applikasi, pada service loadbalancer juga mendapart port range nodeport.
> karena pending kita harus setup metallb terlebih dahulu agar loadbalance dapat diakses menggunakan external ip, sebenarnya kita juga bisa membuat service loadbalancer tanpa setup metallb terlebih dahulu.

- create loadbalancer static external-ip
```
kubectl expose deployment nginx-template --type=LoadBalancer --name=svc-lb-nginx2 --port=80 --target-port=80 --external-ip=10.20.10.201
kubectl get svc
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/be5f5599-6abc-41d6-9b69-3f71c5e7a8cf)

- testing curl service svc-lb-nginx2 dalam cluster 
```
curl 192.164.232.91 -I
curl 10.20.10.201 -I
curl 10.20.10.100:30256 -I
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/c8d2b07c-6c20-4a3f-8495-90fa218fb52e)


- testing curl service svc-lb-nginx2 luar cluster
```
curl 192.164.232.91 -I
curl 10.20.10.201 -I
curl 10.20.10.100:30256 -I
```

![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/5785b519-c241-48ac-b58c-fd8c0f3798d1)
> perhatikan bahwa ketika kita curl dari luar cluster belum bisa, agar dari luar bisa akses ke service tersebut harus menggunakan nodeport. Supaya kita bisa akses ke service menggunakan external-ip kita perlu konfigurasi metallb.

- installation baremetal
```
mkdir metallb && cd metallb
wget https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
kubectl apply -f metallb-native.yaml
```

- set l2advertisment baremetal
```
cat << 'EOF' > l2.yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-lb
  namespace: metallb-system
spec:
  ipAddressPools:
  - 10.20.10.201-10.20.10.240
EOF
kubectl apply -f l2.yaml
kubectl get l2advertisements.metallb.io -n metallb-system
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/b24f5e1d-0010-4863-bc28-c97cd3667efe)

- testing curl di luar cluster
> note
> karena pada service saya sebelumnya restart jadi cluster-ip nya berubah, tapi cluster-ip tetap tidak bisa diakses diluar cluster
```
curl 192.164.232.91 -I
curl 10.20.10.201 -I
curl 10.20.10.100:30256 -I
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/652a7c72-74a6-43ab-9521-ff1f97bba09d)

- fixing pending svc-lb-nginx

![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/7ad2084d-f38c-452b-9f1a-c7e293dd8968)
> penyebab loadbalancing pending karena tidak mendpat ip, agar loadbalancer mendapat ip secara dhcp kita harus konfigurasi metallb ipaddresspool

- set ip address pool
```
cat << 'EOF' > loadbalancing.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lb-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.20.10.149-10.20.10.199
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-nginx
  namespace: metallb-system
spec:
  ipAddressPools:
  - lb-pool # nama ipaddresspool
EOF
kubectl apply -f loadbalancing.yaml
kubectl get svc
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/8685df16-2e30-4aef-be07-4dae0ee4979e)
> note
> untuk loadbalancing svc-lb-nginx juga mendapatkan ip dari lb-pool

- testing curl svc-lb-nginx di luar cluster
```
curl 192.161.145.171 -I
curl 10.20.10.150 -I
curl 10.20.10.100:32093 -I
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/c655cf97-4214-40ef-be96-5508d0fe5a3b)

ExternalName:
Membuat alias untuk layanan eksternal dengan memberikan nama DNS eksternal.Ketika Anda ingin merujuk ke layanan eksternal menggunakan DNS.
