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

disini kita akan mecoba membuat deployment dengan mengubah tampilan nya, jika sebelumnya hanya tampilan default.
- create tampilan web
```
mkdir ~/nginx-template
echo "<h1>Hallo ini adalah nginx-template<h1>" > ~/nginx-template/index.html
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/db27bd46-b9d5-404c-94e5-32f75f19c032)

edit manifest
```
nano nginx_template.yaml
```
edit seperti ini
```

```
