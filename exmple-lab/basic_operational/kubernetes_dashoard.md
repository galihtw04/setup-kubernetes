# Kubernetes Dashboard
> kubernets berfungsi untuk mempermudah kita memanage cluster kubernetes kita, kubernetes dashboard dapat memange resource yang ada pada cluster kita.

deploy kubernetes dashboard.

```
mkdir kube-dashboard && cd kube-dashboard
```

- create namespace
```
kubectl create ns kubernetes-dashboard
```

```
kubectl get ns
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/69ce652a-eda7-41ad-ae15-dc8d179604b2)

- create serviceaccount
<details><summary>user.yaml</summary>

```
cat << 'EOF' > user.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
 name: admin
 namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
 name: admin-user
roleRef:
 apiGroup: rbac.authorization.k8s.io
 kind: ClusterRole
 name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin
  namespace: kubernetes-dashboard
---
apiVersion: v1
kind: Secret
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/service-account.name: admin
type: kubernetes.io/service-account-token
EOF
```
</details>

```
kubectl apply -f user.yaml
sleep 10
kubectl get -n kubernetes-dashboard serviceaccount,secret
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/afa38a82-2f44-430a-b592-c536695178af)

- create serviceaccount, secret, role,clusterrole, clusterrolbinding, & configmaps

<details><summary>serviceaccount.yaml</summary>

```
cat << 'EOF' > serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kubernetes-dashboard
type: Opaque

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-csrf
  namespace: kubernetes-dashboard
type: Opaque
data:
  csrf: ""

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-key-holder
  namespace: kubernetes-dashboard
type: Opaque

---

kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-settings
  namespace: kubernetes-dashboard

---

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
rules:
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs", "kubernetes-dashboard-csrf"]
    verbs: ["get", "update", "delete"]
    # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["kubernetes-dashboard-settings"]
    verbs: ["get", "update"]
    # Allow Dashboard to get metrics.
  - apiGroups: [""]
    resources: ["services"]
    resourceNames: ["heapster", "dashboard-metrics-scraper"]
    verbs: ["proxy"]
  - apiGroups: [""]
    resources: ["services/proxy"]
    resourceNames: ["heapster", "http:heapster:", "https:heapster:", "dashboard-metrics-scraper", "http:dashboard-metrics-scraper"]
    verbs: ["get"]

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
rules:
  # Allow Metrics Scraper to get metrics from the Metrics server
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list", "watch"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard
EOF
```
</details>

```
kubectl apply -f serviceaccount.yaml
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/d90cfcc2-a910-48a8-9919-f3564a55bd42)


- create deployment kube-dashboard

<details><summary>deploymets-dahsboard.yaml</summary>

```
cat << 'EOF' > deploymets-dahsboard.yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      securityContext:
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: kubernetes-dashboard
          image: kubernetesui/dashboard:v2.7.0
          imagePullPolicy: Always
          ports:
            - containerPort: 8443
              protocol: TCP
          args:
            - --auto-generate-certificates
            - --namespace=kubernetes-dashboard
            # Uncomment the following line to manually specify Kubernetes API server Host
            # If not specified, Dashboard will attempt to auto discover the API server and connect
            # to it. Uncomment only if the default does not work.
            # - --apiserver-host=http://my-address:port
          volumeMounts:
            - name: kubernetes-dashboard-certs
              mountPath: /certs
              # Create on-disk volume to store exec logs
            - mountPath: /tmp
              name: tmp-volume
          livenessProbe:
            httpGet:
              scheme: HTTPS
              path: /
              port: 8443
            initialDelaySeconds: 30
            timeoutSeconds: 30
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      volumes:
        - name: kubernetes-dashboard-certs
          secret:
            secretName: kubernetes-dashboard-certs
        - name: tmp-volume
          emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
EOF
```
</details>

```
kubectl apply -f deploymets-dahsboard.yaml
```

```
kubectl get -n kubernetes-dashboard deployments,pods
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/0613c86b-421b-4352-9705-db491b4a77f6)

- create service for kubernetes-dashboard
<details><summary>service-dashboard.yaml</summary>

```
cat << 'EOF' > service-dashboard.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: LoadBalancer
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 32100
  selector:
    k8s-app: kubernetes-dashboard
EOF
```
</details>

```
kubectl apply -f service-dashboard.yaml
```

```
kubectl get svc -n kubernetes-dashboard
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/d5323353-c70e-4c54-8909-fd169ed33a77)

- create deployemnts scrape
<details><summary>deployments-scrape.yaml</summary>

```
cat << 'EOF' > deployments-scrape.yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: dashboard-metrics-scraper
  template:
    metadata:
      labels:
        k8s-app: dashboard-metrics-scraper
    spec:
      securityContext:
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: dashboard-metrics-scraper
          image: kubernetesui/metrics-scraper:v1.0.8
          ports:
            - containerPort: 8000
              protocol: TCP
          livenessProbe:
            httpGet:
              scheme: HTTP
              path: /
              port: 8000
            initialDelaySeconds: 30
            timeoutSeconds: 30
          volumeMounts:
          - mountPath: /tmp
            name: tmp-volume
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      volumes:
        - name: tmp-volume
          emptyDir: {}
EOF
```
</details>

```
kubectl apply -f deployments-scrape.yaml
sleep 20
kubectl get -n kubernetes-dashboard deployments,pods
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/fc154322-3b79-439b-ad5a-6cd3beed934f)

- create service for deployments-scrape
<details><summary>service-scrape.yaml</summary>

```
cat << 'EOF' > service-scrape.yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 8000
      targetPort: 8000
  selector:
    k8s-app: dashboard-metrics-scraper
EOF
```
</details>

```
kubectl apply -f service-scrape.yaml
```

```
kubectl get -n kubernetes-dashboard svc
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/110b3991-3b82-4d3d-8bf2-8e2c244db2fd)

- access kubernetes-dashboard

```
<ip-loadbalancer>
or
<ip-node>:<nodeport>
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/df8e0f9e-55ee-453c-858f-772b62491249)
> untuk token bisa kita cek pada secret admin-user

```
kubectl describe secrets -n kubernetes-dashboard admin-user | grep token
```
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/4f8e8a02-2059-4440-8e3e-69e1a8759825)
> copy token dan paster pada dashboard kubernetes
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/85ff89cc-ea6d-4164-8f38-4b4f17e34039)

