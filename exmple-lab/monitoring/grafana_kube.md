# deploy grafana on kubernetes
> Grafana adalah platform open-source yang digunakan untuk membuat dan mengelola dashboard monitoring dan analisis data. Fungsi utama Grafana adalah menyajikan data secara visual melalui grafik dan dashboard yang interaktif.

- create pvc
 > note di dalam cluster sudah ada nfs provisioner untuk pv

```
mkdir ~/grafana && cd ~/grafana
```

<details><summary>storage-pvc.yaml</summary>

```
cat << 'EOF' > storage-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: monitoring
  name: grafana-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
EOF
```
</details>

```
kubectl apply -f storage-pvc.yaml
```

- create deployment

<details><summary>deplyemnt.yaml</summary>

```
cat << 'EOF' > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: grafana
  namespace: monitoring
  name: grafana
spec:
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      securityContext:
        fsGroup: 472
        supplementalGroups:
          - 0
      containers:
        - name: grafana
          image: grafana/grafana:8.4.4
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
              name: http-grafana
              protocol: TCP
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /robots.txt
              port: 3000
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 30
            successThreshold: 1
            timeoutSeconds: 2
          livenessProbe:
            failureThreshold: 3
            initialDelaySeconds: 30
            periodSeconds: 10
            successThreshold: 1
            tcpSocket:
              port: 3000
            timeoutSeconds: 1
          resources:
            requests:
              cpu: 250m
              memory: 750Mi
          volumeMounts:
            - mountPath: /mnt/grafana
              name: grafana-pv
      volumes:
        - name: grafana-pv
          persistentVolumeClaim:
            claimName: grafana-pvc
EOF
```
</details>

```
kubectl apply -f deployment.yaml
```

- create service

<details><summary>service.yaml</summary>

```
cat << 'EOF' > service.yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  ports:
    - port: 3000
      protocol: TCP
      targetPort: http-grafana
  selector:
    app: grafana
  sessionAffinity: None
  type: NodePort
EOF
```
</details>

```
kubectl apply -f service.yaml
```

![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/75f3b3f6-94e5-4a14-b790-e064bbef05f4)
![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/5d6e17d7-26c2-4820-b329-50d38f5be618)
