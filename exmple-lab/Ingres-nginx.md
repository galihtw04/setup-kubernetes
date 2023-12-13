# Setup Ingress nginx
refrensi : https://github.com/kubernetes/ingress-nginx

install ingress Controller
> set label agar nanti kita akan deploy controllernya hanya pada worker dengan label app.kubernetes.io/ingress="true"
```
kubectl label nodes k8s-student-worker01 app.kubernetes.io/ingress="true"
kubectl label nodes k8s-student-worker02 app.kubernetes.io/ingress="true"
```

check label
```
kubectl get nodes --show-labels
kubectl describe node k8s-student-worker01
kubectl describe node k8s-student-worker02
```

![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/7873b773-cb4e-4711-b3f4-40820bcf9d8b)
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/3c4f3aa1-663e-4b24-ae20-4cad2b6c3e03)

