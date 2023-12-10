# namespace
konsep yang digunakan untuk memisahkan dan mengorganisir sumber daya di dalam sebuah kluster, sederhana kita dapat mengelompokan seperti service,pods,deployment, dan rescource lainnya. 

check namespace
```
kubectl get namespace
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/b9c61d20-0b17-47b2-a0af-74ea7c5c2881)

check resource
```
kubectl get -n default all
kubectl get -n kube-node-lease all
kubectl get -n kube-public all
kubectl get -n kube-system all
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/2b04c256-51f7-4b1d-9e91-541824f6a2e8)
> ketika kita create deployment atau resource lain dan tidak mendefinisikan namaspace maka akan ditaruh di namespace default

![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/098bd855-23a7-474d-ad4a-28b55aac4507)
> tidak ada resource pada namespace kube-node-lease dan kube-public

![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/405aa527-8fd9-4964-8c1f-9bb21fe5235e)

![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/d5261388-114b-4972-95a6-66a6d1218f3c)



