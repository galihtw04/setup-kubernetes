# Setup Ingress nginx
refrensi : https://github.com/kubernetes/ingress-nginx


install ingress Controller
 - apply with manifest
```
mkdir Ingress && cd ingress
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/cloud/deploy.yaml
kubectl apply -f deploy.yaml
```
- check
```
kubectl get all -n ingress-nginx
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/c9d05be3-7508-4a4c-bf2b-22ad7b3dacfa)

scalae controller
```
kubectl scale --replicas 2 -n ingress-nginx deployment ingress-nginx-controller
kubectl get all -n ingress-nginx
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/614f7a39-4f3b-43bf-aa46-9d10c1426f39)

install ingress-nginx with helm
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

install helm
```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```


create daemonset for Ingress nginx
```
cat << 'EOF' >> daemonset.yaml
controller:
  kind: DaemonSet
  metrics:
    port: 10254
    enabled: true
  podAnnotations:
    prometheus.io/scrape: true
    prometheus.io/port: 10254
  extraArgs:
    default-server-port: 8119
  service:
    enabled: true
    type: NodePort
    nodePorts:
      http: 31080
      https: 31443
  nodeSelector:
    app.kubernetes.io/ingress: "true"
  tolerations:
    - operator: Exists
EOF
```

install Ingress-nginx
```
kubectl delete -f deploy.yaml # eksekusi jika sudah install ingress menggunakan manifest
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo list
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/92cc9a6c-1ec9-49a8-b9f8-f2aa8d38f0fe)

```
helm install ingress-nginx ingress-nginx  --values daemonset.yaml --namespace ingress-nginx --create-namespace
kubectl get all -n ingress-nginx -o wide
```

testing
```
cat << 'EOF' > testing.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: pods-testing
    spec:
      containers:
      - name: pods-testing
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: pods-testing
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: nginx-test.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
EOF
kubectl apply -f testing
```
