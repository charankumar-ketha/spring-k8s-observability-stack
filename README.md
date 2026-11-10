# Production-Grade Spring Boot Observability Stack on Kubernetes

A containerized, resilient microservices architecture deployed on an AWS EC2 instance using a Minikube Kubernetes cluster. The stack features automated container orchestration, dynamic persistent storage, health-probe auto-recovery, and full-tier monitoring with Prometheus and Grafana.

---

## Architecture Overview

```
                      Internet / Browser Traffic
                                  │
                                  ▼
                    AWS EC2 Public IP (Port Access)
                    ┌───────────────────────────────┐
                    │  Security Group Configuration │
                    │  Ports: 22, 8080, 9090, 3000  │
                    └──────────────┬────────────────┘
                                   │
              ┌────────────────────┴────────────────────┐
              │ Minikube Cluster (Container Network)    │
              │                                         │
              │   ┌──────────────────────────────────┐  │
              │   │     spring-service (NodePort)    │  │
              │   │           Port: 8080             │  │
              │   └───────────────┬──────────────────┘  │
              │                   │                     │
              │       ┌───────────┴───────────┐         │
              │       ▼                       ▼         │
              │ ┌──────────────┐        ┌─────────────┐ │
              │ │ Spring Boot  │        │ Spring Boot │ │
              │ │ Pod (Repl 1) │        │ Pod(Repl 2) │ │
              │ └──────┬───────┘        └──────┬──────┘ │
              │        │                       │        │
              │        └───────────┬───────────┘        │
              │                    │ /actuator/prometheus
              │                    ▼                    │
              │        ┌────────────────────────┐       │
              │        │  Prometheus Pod & Svc  │       │
              │        │       Port: 9090       │       │
              │        └───────────┬────────────┘       │
              │                    │ Scrape Metrics     │
              │                    ▼                    │
              │        ┌────────────────────────┐       │
              │        │   Grafana Pod & Svc    │       │
              │        │       Port: 3000       │       │
              │        └────────────────────────┘       │
              │                    │                    │
              │                    ▼                    │
              │        ┌────────────────────────┐       │
              │        │  PostgreSQL (ClusterIP)│       │
              │        │       Port: 5432       │       │
              │        └───────────┬────────────┘       │
              │                    │                    │
              │                    ▼                    │
              │        ┌────────────────────────┐       │
              │        │    postgres-pvc 1Gi    │       │
              │        │   (Persistent Storage) │       │
              │        └────────────────────────┘       │
              └─────────────────────────────────────────┘
```

---

## Tech Stack & Core Components

| Component | Technology | Role & Configuration |
| :--- | :--- | :--- |
| **Backend API** | Spring Boot 3 / Java 17 | Multi-replica REST service instrumented with Spring Actuator and Micrometer. |
| **Database** | PostgreSQL 15 | Relational persistence service isolated inside internal cluster networking (`ClusterIP`). |
| **Storage Engine**| Kubernetes PVC (`standard`) | 1Gi dynamic volume claim mounted to `/var/lib/postgresql/data` for crash resistance. |
| **Monitoring** | Prometheus v2.51.0 | Real-time scraper collecting metrics at 5-second intervals via a custom ConfigMap. |
| **Visualization**| Grafana 10.4.0 | Dashboard #4701 visualizing JVM heap allocation, active threads, and HTTP latency. |
| **Infrastructure**| AWS EC2 & Minikube | Ubuntu host running Docker container runtime and Minikube Kubernetes node. |

---

## Repository Structure

```
├── backend/
│   ├── src/                    # Spring Boot application source code
│   ├── pom.xml                 # Maven dependencies & plugins
│   └── Dockerfile              # Multi-stage build targeting distroless/alpine JRE
├── deployment.yaml             # Kubernetes Deployments & PersistentVolumeClaim
├── service.yaml                # ClusterIP and NodePort Service definitions
├── prometheus-config.yaml      # Prometheus ConfigMap with custom scrape rules & labels
├── start.sh                    # Automated boot, rollout check, and port-forward script
├── .gitignore                  # Exclusion file preventing leaks of logs and binaries
└── README.md                   # Complete architectural & operational guide
```

---

## Detailed Manifest Specifications

### 1. Multi-Stage Container Packaging (`backend/Dockerfile`)
Uses a two-stage build to isolate Maven compilation dependencies and minimize production image attack surface:

```dockerfile
# Stage 1: Build JAR using Maven
FROM maven:3.9.6-eclipse-temurin-17-alpine AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Minimal Distroless / Alpine JRE Runtime
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=builder /app/target/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["java", "-XX:+UseG1GC", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

---

### 2. Workload & Storage Configuration (`deployment.yaml`)
Handles stateful data persistence, backend scaling (2 replicas), and Prometheus configuration volume mounts:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deployment
  labels:
    app: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: "proddb"
            - name: POSTGRES_USER
              value: "postgres"
            - name: POSTGRES_PASSWORD
              value: "postgres123"
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgres-storage
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-pvc
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-deployment
  labels:
    app: spring-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: spring-app
  template:
    metadata:
      labels:
        app: spring-app
    spec:
      containers:
        - name: spring-app
          image: charankumarketha/production-spring-backend:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_DATASOURCE_URL
              value: "jdbc:postgresql://postgres-service:5432/proddb"
            - name: SPRING_DATASOURCE_USERNAME
              value: "postgres"
            - name: SPRING_DATASOURCE_PASSWORD
              value: "postgres123"
            - name: SPRING_JPA_HIBERNATE_DDL_AUTO
              value: "update"
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 10
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-deployment
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus:v2.51.0
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: config-volume
              mountPath: /etc/prometheus/prometheus.yml
              subPath: prometheus.yml
      volumes:
        - name: config-volume
          configMap:
            name: prometheus-config
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana-deployment
  labels:
    app: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:10.4.0
          ports:
            - containerPort: 3000
```

---

### 3. Service Routing (`service.yaml`)
Enforces least-privilege networking by locking PostgreSQL to `ClusterIP` and exposing public services via `NodePort`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: spring-service
spec:
  type: NodePort
  selector:
    app: spring-app
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 30080
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
spec:
  type: NodePort
  selector:
    app: prometheus
  ports:
    - protocol: TCP
      port: 9090
      targetPort: 9090
      nodePort: 30090
---
apiVersion: v1
kind: Service
metadata:
  name: grafana-service
spec:
  type: NodePort
  selector:
    app: grafana
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
      nodePort: 30300
```

---

### 4. Scrape Pipeline Configuration (`prometheus-config.yaml`)
Applies target labeling so Prometheus automatically decorates metrics with `application: spring-actuator`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 5s
      evaluation_interval: 5s

    scrape_configs:
      - job_name: 'spring-actuator'
        metrics_path: '/actuator/prometheus'
        static_configs:
          - targets: ['spring-service:8080']
            labels:
              application: 'spring-actuator'
```

---

## Automation: One-Click Startup & Recovery Script (`start.sh`)

The included `start.sh` boots Minikube, synchronizes all manifests, awaits ready states, and runs self-healing background port-forward loops.

```bash
#!/usr/bin/env bash
set -e

DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
cd "$DIR"

echo "=========================================="
echo " Starting Minikube & Kubernetes Microservices "
echo "=========================================="

# 1. Start Minikube if inactive
if ! minikube status | grep -q "Running"; then
    echo "[+] Starting Minikube..."
    minikube start --driver=docker
else
    echo "[✓] Minikube is already running."
fi

# 2. Synchronize configs and manifests
echo "[+] Applying ConfigMaps, Services, and Deployments..."
kubectl apply -f prometheus-config.yaml
kubectl apply -f service.yaml
kubectl apply -f deployment.yaml

# 3. Wait for pod rollouts
echo "[+] Waiting for workloads to be ready..."
kubectl rollout status deployment/postgres-deployment --timeout=90s
kubectl rollout status deployment/spring-deployment --timeout=120s
kubectl rollout status deployment/prometheus-deployment --timeout=90s
kubectl rollout status deployment/grafana-deployment --timeout=90s

# 4. Clean stale port-forwards
echo "[+] Cleaning up stale port-forwarding listeners..."
pkill -f "kubectl port-forward" 2>/dev/null || true
sleep 1

# 5. Launch resilient port-forwards (0.0.0.0)
echo "[+] Spawning resilient port-forward loops..."
nohup bash -c 'while true; do kubectl port-forward --address 0.0.0.0 svc/spring-service 8080:8080; sleep 2; done' > "$DIR/spring-pf.log" 2>&1 &
nohup bash -c 'while true; do kubectl port-forward --address 0.0.0.0 svc/prometheus-service 9090:9090; sleep 2; done' > "$DIR/prom-pf.log" 2>&1 &
nohup bash -c 'while true; do kubectl port-forward --address 0.0.0.0 svc/grafana-service 3000:3000; sleep 2; done' > "$DIR/grafana-pf.log" 2>&1 &

sleep 3

PUBLIC_IP=$(curl -s [http://checkip.amazonaws.com](http://checkip.amazonaws.com) || curl -s ifconfig.me || echo "localhost")

echo "=========================================="
echo " Cluster stack is healthy and forwarding! "
echo "=========================================="
echo " - Spring Boot API : http://${PUBLIC_IP}:8080/api/health"
echo " - Prometheus      : http://${PUBLIC_IP}:9090/targets"
echo " - Grafana         : http://${PUBLIC_IP}:3000"
echo "=========================================="
```

---

## Verification & Validation Proofs

### 1. Cluster Status Output
```bash
$ kubectl get pods,svc,pvc -o wide
NAME                                            READY   STATUS    RESTARTS   AGE   IP            NODE
pod/grafana-deployment-8596d594f9-pqw9m         1/1     Running   0          49m   10.244.0.6    minikube
pod/postgres-deployment-5545d9cdbc-v6xnd        1/1     Running   0          49m   10.244.0.5    minikube
pod/prometheus-deployment-77df9f7654-b4c9m      1/1     Running   0          7m    10.244.0.10   minikube
pod/spring-deployment-78cd7fff75-9gm8l          1/1     Running   1          49m   10.244.0.3    minikube
pod/spring-deployment-78cd7fff75-m8x7z          1/1     Running   0          49m   10.244.0.7    minikube

NAME                         TYPE        CLUSTER-IP       PORT(S)          AGE   SELECTOR
service/grafana-service      NodePort    10.102.213.224   3000:30300/TCP   49m   app=grafana
service/postgres-service     ClusterIP   10.99.242.124    5432/TCP         49m   app=postgres
service/prometheus-service   NodePort    10.106.28.196    9090:30090/TCP   49m   app=prometheus
service/spring-service       NodePort    10.106.143.113   8080:30080/TCP   49m   app=spring-app

NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
persistentvolumeclaim/postgres-pvc  Bound    pvc-ef827979-a394-4f12-833c-7a5ff78d3abf   1Gi        RWO            standard
```

### 2. End-to-End Persistence Validation
```bash
# Insert a new record into PostgreSQL through Spring Boot API
$ curl -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{"name": "Mechanical Keyboard", "price": 89.99}'
{"id":1,"name":"Mechanical Keyboard","price":89.99}

# Fetch the persisted record
$ curl http://localhost:8080/api/products
[{"id":1,"name":"Mechanical Keyboard","price":89.99}]
```

### 3. Production Monitoring Verification (Grafana Dashboard #4701)
* **Application Scope:** Scraped target `spring-actuator` mapped to `spring-service:8080`.
* **Telemetry Collected:**
  * **Heap Usage:** Steady state at ~1.42% (G1 Garbage Collector).
  * **Non-Heap Usage:** Metaspace and Native Thread memory at ~7.95%.
  * **Throughput:** Real-time request monitoring with sub-5ms response duration.

---

## Clean Shutdown & Cost Management

To pause resources and eliminate cloud hosting charges without losing configurations or persistent volume state:

```bash
# Stop Minikube cluster cleanly
minikube stop

# From local workstation or AWS Console:
# Stop EC2 instance (Do NOT Terminate)
aws ec2 stop-instances --instance-ids <YOUR-INSTANCE-ID>
```
