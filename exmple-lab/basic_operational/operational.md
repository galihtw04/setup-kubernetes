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


Resource kubernetes

- Node:
Node merupakan mesin fisik atau virtual yang menjalankan aplikasi Kubernetes dan dapat berisi beberapa pod. Setiap node berisi kubelet, yang bertanggung jawab untuk berkomunikasi dengan master dan menjalankan pod pada node tersebut.

- Pod:
Pod adalah unit terkecil dalam Kubernetes dan merupakan lingkungan di mana kontainer berjalan. Pod dapat berisi satu atau beberapa kontainer yang saling berbagi sumber daya dan jaringan. Mereka berkomunikasi satu sama lain menggunakan localhost.

- ReplicaSet:
ReplicaSet adalah sumber daya yang memastikan bahwa jumlah tertentu dari pod (replika) selalu berjalan. Jika ada pod yang gagal atau dihapus, ReplicaSet akan membuat pod baru untuk menggantikannya sehingga jumlah replika tetap konsisten.

- Deployment:
Deployment menyediakan cara deklaratif untuk mendefinisikan aplikasi Kubernetes. Ini memungkinkan Anda mendefinisikan berapa banyak pod dari suatu aplikasi yang harus berjalan dan memungkinkan pembaruan yang mudah dengan menjalankan versi baru sambil menjaga ketersediaan aplikasi.

- Service:
Service memungkinkan pod untuk ditemukan satu sama lain dan berkomunikasi di dalam cluster Kubernetes. Service memberikan alamat IP dan nama DNS yang konstan untuk satu atau beberapa pod, memungkinkan aplikasi untuk berkomunikasi dengan pod lain tanpa harus mengetahui detail internal dari setiap pod.

- ConfigMap dan Secret:
ConfigMap dan Secret adalah cara untuk mengelola konfigurasi dan data rahasia dalam Kubernetes. ConfigMap dapat menyimpan konfigurasi yang tidak bersifat rahasia, sementara Secret digunakan untuk menyimpan informasi yang bersifat rahasia, seperti kata sandi atau kunci API.

- Ingress:
Ingress adalah sumber daya yang memungkinkan pengelolaan akses eksternal ke layanan dalam cluster. Ini memungkinkan routing lalu lintas eksternal ke layanan-layanan yang sesuai dalam cluster berdasarkan aturan tertentu.

- Persistent Volumes (PV) dan Persistent Volume Claims (PVC):
PV dan PVC memungkinkan menyimpan data yang persisten dalam lingkungan kontainer. PV adalah penyimpanan yang bersifat persisten dan independen, sedangkan PVC adalah permintaan pengguna untuk penyimpanan yang persisten.

- StatefulSets:
StatefulSets digunakan untuk aplikasi yang memerlukan identitas yang unik dan persisten untuk setiap instansinya, seperti basis data. Mereka memastikan urutan penggunaan nama dan identitas yang persisten untuk setiap pod.
