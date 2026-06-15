# Local Kubernetes Deployment Pipeline: Scalable Containerized Microservice

This repository demonstrates the end-to-end containerization, local orchestration, and performance validation of a microservice using **Docker**, **Minikube (Multi-Node)**, and **Kubernetes**. It replicates modern cloud-native MLOps and engineering workflows in a local environment, focusing on infrastructure resiliency, zero-downtime updates, and load testing.

---

## Architecture & Core Workflow

The pipeline bridges local development with production-grade orchestration principles. It follows a multi-stage execution model:

```text
[ Source Code ] ──> [ Multi-Stage Docker Build ] ──> [ Local Minikube Cluster (2 Nodes) ]
                                                                 │
                                                    ┌────────────┴────────────┐
                                                    ▼                         ▼
                                         [ Pod 1: Replicated ]     [ Pod 2: Replicated ]
                                                    ▲                         ▲
                                                    └────────────┬────────────┘
                                                                 │
                                                      [ K8s ClusterIP/NodePort ] 
                                                                 ▲
                                                                 │
                                                    [ Postman Load Testing ]

```

---

## Engineered Competencies Demonstrated

* **Multi-Node Cluster Orchestration**: Configured and managed a 2-node local Kubernetes topology via Minikube to simulate horizontal scaling and fault tolerance across discrete worker nodes.
* **Resilient Lifecycle Management**: Authored manifests incorporating native declarative recovery strategies, health checks, and declarative scaling.
* **Performance & Scale Validation**: Utilized Postman performance suites to run fixed-configuration stress testing (10 concurrent virtual users for 1 minute) to evaluate service reliability and threshold limits under load.
* **Advanced Troubleshooting & Network Triage**: Documented automated workarounds for air-gapped/proxy environments (`registry.k8s.io` connection failures) and private registry orchestration (`ImagePullBackOff` mitigations).

---

## Production-Grade Manifests

The declarative infrastructure state is managed via explicit configurations designed for high availability.

### `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubernetes-test-app
  labels:
    app: test-app
    tier: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
      - name: test-app-container
        image: kubernetes-test-app:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "250m"
            memory: "256Mi"
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 15
          periodSeconds: 20
        readinessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 10

```

### `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubernetes-test-app-service
spec:
  type: NodePort
  selector:
    app: test-app
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
      nodePort: 30080

```

---

## Step-by-Step Execution Guide

### 1. Containerization & Local Verification

Build and test the application image inside the host Docker daemon to ensure structural integrity and functional correctness.

```bash
# Build the Docker image
docker build -t kubernetes-test-app:latest .

# Verify image registry locally
docker images

# Execute local container to test exposed ports
docker run -p 5000:5000 kubernetes-test-app:latest

```

### 2. Multi-Node Cluster Initialization

Spin up a multi-node Minikube cluster using embedded certificates to establish a robust environment mimicking a public cloud topology.

```bash
# Initialize cluster with 2 worker nodes and embedded certificates
minikube start --nodes=2 --embed-certs

# Confirm cluster state and resource availability across nodes
minikube status
kubectl get nodes -A

```

### 3. Image Sideloading into Kubernetes Context

To bypass external registry overhead during local staging, inject the image directly into the Minikube runtime cache.

```bash
# Sideload the local container image into the internal cluster cache
minikube image load kubernetes-test-app:latest

# Verify the image is present inside the cluster environment
minikube image list

```

### 4. Workload Deployment & Verification

Apply the declarative state configurations and evaluate active pod statuses across the cluster topology.

```bash
# Apply deployment configurations
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# Monitor deployment progress and verify multi-node distribution
kubectl get pods -o wide
kubectl get service

```

### 5. Routing Traffic & Observability

Expose the service to the host network interface and open the unified orchestration control panel.

```bash
# Expose the application endpoint securely
minikube service kubernetes-test-app-service --url

# Launch the interactive Kubernetes Dashboard for real-time cluster monitoring
minikube dashboard

```

---

## Operational Runbook & Edge-Case Troubleshooting

### Problem: Host Proxy/DNS Resolution Failures

When setting up clusters under strict firewalls or corporate proxies, the control plane may fail to reach `registry.k8s.io`.

**Resolution Strategies:**

1. **Explicit Proxy Configuration**: Inject networking environment flags explicitly into the daemon runtime during setup:
```bash
minikube start --docker-env HTTP_PROXY=http://<proxy-ip>:<port> --docker-env HTTPS_PROXY=https://<proxy-ip>:<port>

```


2. **Environment Purge**: If proxies are causing unexpected loopbacks, clear active host variables before clean initialization:
```bash
unset HTTP_PROXY; unset HTTPS_PROXY; unset NO_PROXY
minikube stop && minikube delete --all

```



### Problem: `ImagePullBackOff` Failures

Occurs when kubelet cannot resolve the image identifier within local scopes or lacks downstream authentication details for public/private platforms.

**Resolution Strategies:**

1. **Image Pull Policy Modification**: Ensure the manifest contains `imagePullPolicy: IfNotPresent` or `Never` when using local image side-loading.
2. **Upstream Remote Registry Failover**: Re-tag and map the container to an authenticated, public repository stream:
```bash
# Re-tag local runtime context
docker tag kubernetes-test-app:latest <your-dockerhub-username>/kubernetes-test-app:latest

# Push image to upstream repository
docker push <your-dockerhub-username>/kubernetes-test-app:latest

# Update deployment manifest image field to point directly to remote repository
kubectl set image deployment/kubernetes-test-app test-app-container=<your-dockerhub-username>/kubernetes-test-app:latest

```



---

## Automated Verification & Stress Testing

To validate service resilience and auto-healing capabilities under systemic stress, a synthetic traffic workload was executed using Postman's Performance Module.

* **Test Criteria**: 10 Concurrent Users | Fixed Continuous Iterations | 60 Second Duration
* **Observed Metrics Evaluated**: Error rate percentage, mean response latencies, and container failover stability.
* **Resilience Assertion**: When simulated instances were manually killed (`kubectl delete pod <pod-id>`) midway through the execution window, the replication controller automatically provisioned replacement pods onto the secondary node without compromising target availability constraints or breaching error limits.

```bash
# Monitor container failover logs in real-time during load testing
kubectl logs -f -l app=test-app --max-log-requests=2

```
