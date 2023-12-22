# deploy monitoring using prometheus and grafana

# # deploy prometheus
- create directory
```
mkdir prometheus && cd prometheus
```

- create namespace
<details><summary>namespace.yaml</summary>

```
cat << 'EOF' > namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
EOF
```
</details>

- create clusterrole and service account

<details><summary>clusterrole.yaml</summary>

```
cat << 'EOF' > clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - configmaps
  - endpoints
  - pods
  - ingresses
  - namespaces
  verbs: ["get", "list", "watch", "exec"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: default
  namespace: monitoring
EOF
```
</details>

- create configmap
> configmap ini digunakan untuk menentukan target yang akan diambil metricnya, dan confgimap ini akan di mount di deployment

<details><summary>configmaps.yaml</summary>

```
cat << 'EOF' > configmaps.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-server-conf
  labels:
    name: prometheus-server-conf
  namespace: monitoring
data:
  prometheus.rules: |-
    groups:
    - name: ops-monit demo alert
      rules:
      - alert: High Pod Memory
        expr: sum(container_memory_usage_bytes) > 1
        for: 1m
        labels:
          severity: slack
        annotations:
          summary: High Memory Usage
  prometheus.yml: |-
    global:
      scrape_interval: 5s
      evaluation_interval: 5s
    rule_files:
      - /etc/prometheus/prometheus.rules
    alerting:
      alertmanagers:
      - scheme: http
        static_configs:
        - targets:
          - "alertmanager.monitoring.svc:9093"

    scrape_configs:
      - job_name: 'node-exporter'
        kubernetes_sd_configs:
          - role: endpoints
        relabel_configs:
        - source_labels: [__meta_kubernetes_endpoints_name]
          regex: 'node-exporter'
          action: keep


      - job_name: 'kube-etcd'
        kubernetes_sd_configs:
        - role: pod
        scheme: https
        tls_config:
          insecure_skip_verify: true
          cert_file: /opt/prometheus/secrets/apiserver-etcd-client.crt
          key_file: /opt/prometheus/secrets/apiserver-etcd-client.key
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_label_component]
          action: keep
          regex: etcd
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:2379
          target_label: __address__
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: node
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: pod
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)

      - job_name: 'kubernetes-api'
        kubernetes_sd_configs:
        - role: pod
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_label_component]
          action: keep
          regex: kube-apiserver
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:6443
          target_label: __address__
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: node
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: pod


      - job_name: 'kube-state-metricsv2'
        kubernetes_sd_configs:
        - role: pod
        scheme: http
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
          action: keep
          regex: kube-state-metrics
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: pod
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: namespace
        - source_labels: [__meta_kubernetes_pod_node_name]
          action: replace
          target_label: nodev2

      - job_name: 'node-exporter-service'
        static_configs:
        - targets:
          - '10.20.10.10:9100'
          - '10.20.10.11:9100'
          - '10.20.10.12:9100'
          - '10.20.10.13:9100'
          - '10.20.10.14:9100'
        metrics_path: '/metrics'

      - job_name: 'kubernetes-scheduler-type2'
        kubernetes_sd_configs:
        - role: pod
        scheme: https
        tls_config:
                insecure_skip_verify: true
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_label_component]
          action: keep
          regex: kube-scheduler
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: node
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_name_(.+)

      - job_name: 'kubernetes-nodes'

        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
        - role: node

        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics

      - job_name: 'kubernetes-manager'
        kubernetes_sd_configs:
        - role: pod
        scheme: https
        tls_config:
                insecure_skip_verify: true
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: kube
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)

      - job_name: 'kubernetes-pods'

        kubernetes_sd_configs:
        - role: pod

        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name
        - source_labels: [__meta_kubernetes_node_name]
          action: replace
          target_label: worker

      - job_name: 'kubernetes-cadvisor'

        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
        - role: node
        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
        - source_labels: [__meta_kubernetes_node_name]
          action: replace
          target_label: node

      - job_name: 'kubernetes-service-endpoints'

        kubernetes_sd_configs:
        - role: endpoints

        relabel_configs:
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
          action: replace
          target_label: __scheme__
          regex: (https?)
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
          action: replace
          target_label: __address__
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
        - action: labelmap
          regex: __meta_kubernetes_service_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_service_name]
          action: replace
          target_label: kubernetes_name
EOF
```
</details>

- create secret to etcd metrics
> agar kita bisa menambahkan etcd ke prometheus kita perlu membuat secret untuk hit etcd

curl manual
```
curl https://10.20.10.10:2379/metrics --cert /etc/kubernetes/pki/apiserver-etcd-client.crt --key /etc/kubernetes/pki/apiserver-etcd-client.key -k
```

create secret
<details><summary>secret</summary>

```
kubectl -n monitoring create secret generic etcd-ca --from-file=apiserver-etcd-client.key --from-file apiserver-etcd-client.crt
```
</details>

- create deployment

<details><summary>deployment.yaml</summary>

```
cat << 'EOF' > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-deployment
  namespace: monitoring
  labels:
    app: prometheus-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-server
  template:
    metadata:
      labels:
        app: prometheus-server
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus
          args:
            - "--storage.tsdb.retention.time=31d"
            - "--config.file=/etc/prometheus/prometheus.yml"
            - "--storage.tsdb.path=/prometheus/"
            - "--web.enable-lifecycle"
          env:
            - name: NO_PROXY
              value: .svc,.local,127.0.0.1,10.20.10.10,10.20.10.11,10.20.10.12,10.20.10.100
          ports:
            - containerPort: 9090
          volumeMounts:
            - mountPath: /opt/prometheus/secrets
              name: etcd-ca
            - name: prometheus-config-volume
              mountPath: /etc/prometheus/
            - name: prometheus-storage-volume
              mountPath: /prometheus/
        - name: configmap-reload
          image: "jimmidyson/configmap-reload:latest"
          imagePullPolicy: "IfNotPresent"
          securityContext:
            runAsNonRoot: true
            runAsUser: 65534
          env:
          - name: NO_PROXY
            value: localhost,127.0.0.1
          args:
            - --volume-dir=/etc/prometheus/
            - --webhook-url=http://127.0.0.1:9090/-/reload
          volumeMounts:
            - mountPath: /opt/prometheus/secrets
              name: etcd-ca
            - mountPath: /etc/prometheus/
              name: prometheus-config-volume
              readOnly: true
      volumes:
        - name: etcd-ca
          secret:
            defaultMode: 420
            secretName: etcd-ca
        - name: prometheus-config-volume
          configMap:
            defaultMode: 420
            name: prometheus-server-conf
        - name: prometheus-storage-volume
          emptyDir: {}
EOF
```
</details>

- create service nodeport
<details><summary>service.yaml</summary>

```
cat << 'EOF' > service.yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  namespace: monitoring
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '9090'
spec:
  selector: 
    app: prometheus-server
  type: NodePort  
  ports:
    - port: 8080
      targetPort: 9090 
      nodePort: 30000
EOF
```
</details>

apply
```
kubectl apply -f namespace.yaml
kubectl apply -f clusterrole.yaml
kubectl apply -f configmaps.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

![image](https://github.com/galihtw04/setup-kubernetes/assets/96242740/31689596-f34d-4db4-aa9c-aa295084646c)

