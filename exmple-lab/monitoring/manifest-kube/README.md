# Edited manifest
> Ada beberapa manifest yang perlu kita edited agar metric nya keluar, dan pada prometheus dapat metricnya

disini kita perlu edit pada semua node master
- edit kube-controller-manager.yaml

```
nano /etc/kubernetes/manifests/kube-controller-manager.yaml
```

```
## ----
  name: kube-controller-manager
  namespace: kube-system
  annotations: # add anotations
      prometheus.io/scrape: 'kube'  # add 
      prometheus.io/port: '10257' # add
## -----
    - kube-controller-manager
    - --allocate-node-cidrs=true
    - --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --bind-address=0.0.0.0 # edited
```

- edit kube-apiserver.yaml

```
nano /etc/kubernetes/manifests/kube-apiserver.yaml
```

```
## ---
  name: kube-apiserver
  namespace: kube-system
  annotations: # add anotations
      prometheus.io/port: '6443' # add
## ---
```

- edit https://10.20.10.10:2380

```
nano /etc/kubernetes/manifests/kube-scheduler.yaml
```

```
## ----
  name: kube-scheduler
  namespace: kube-system
  annotations: # add anotations
    prometheus.io/port: "10259" # add
    prometheus.io/path: "/metrics" # add
## ----
  containers:
  - command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=0.0.0.0 # edited
    - --kubeconfig=/etc/kubernetes/scheduler.conf
## ---
```

> edited pada manifest diatas dari kube-controller-manager.yaml,kube-apiserver.yaml, dan kube-scheduler.yaml pastikan di eksekusi di semua node master

restart pods manual/static
 - kube-apiserver
```
kubectl delete -n kube-system kube-apiserver-k8s-student-master01
kubectl delete -n kube-system kube-apiserver-k8s-student-master02
kubectl delete -n kube-system kube-apiserver-k8s-student-master03
```

- kube-controller
```
kubectl delete -n kube-system kube-controller-manager-k8s-student-master01
kubectl delete -n kube-system kube-controller-manager-k8s-student-master02
kubectl delete -n kube-system kube-controller-manager-k8s-student-master03
```

- kube-scheduler
```
kubectl delete -n kube-system kube-scheduler-k8s-student-master01
kubectl delete -n kube-system kube-scheduler-k8s-student-master02
kubectl delete -n kube-system kube-scheduler-k8s-student-master03
```
> pastikan semua pods yang di delete recreate

```
kubectl get pods -n kube-system
```
