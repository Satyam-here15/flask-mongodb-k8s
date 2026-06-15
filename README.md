# Flask MongoDB Kubernetes Deployment

A Python Flask application connected to MongoDB, fully deployed on a Kubernetes cluster using Minikube. This project demonstrates containerization, orchestration, autoscaling, persistent storage, and database authentication.

---

## Project Structure

flask-mongodb-k8s/

├── app/

│   ├── app.py

│   ├── requirements.txt

│   └── Dockerfile

├── k8s/

│   ├── mongo-secret.yaml

│   ├── mongo-pv.yaml

│   ├── mongo-pvc.yaml

│   ├── mongo-statefulset.yaml

│   ├── mongo-service.yaml

│   ├── flask-deployment.yaml

│   ├── flask-service.yaml

│   └── flask-hpa.yaml

└── README.md

---

## Prerequisites

Make sure the following are installed on your system:

- Python 3.8 or later
- Docker Desktop
- Minikube
- kubectl

---

## Part 1: Docker Setup

### 1. Build the Docker Image

```bash
docker build -t satyam150703/flask-mongo-app:v1 ./app
```

### 2. Push the Image to Docker Hub

```bash
docker login
docker push satyam150703/flask-mongo-app:v1
```

The image is publicly available at:
`docker.io/satyam150703/flask-mongo-app:v1`

---

## Part 2: Kubernetes Deployment on Minikube

### 1. Start Minikube

```bash
minikube start --driver=docker
```

### 2. Deploy All Resources at Once

```bash
kubectl apply -f k8s/
```

This creates the following resources:
- MongoDB Secret (credentials)
- Persistent Volume and Persistent Volume Claim
- MongoDB StatefulSet with authentication
- MongoDB ClusterIP Service
- Flask Deployment with 2 replicas
- Flask NodePort Service
- Horizontal Pod Autoscaler

### 3. Verify Everything is Running

```bash
kubectl get pods
kubectl get svc
kubectl get pv
kubectl get pvc
kubectl get hpa
```

Expected output for pods:
NAME                        READY   STATUS    RESTARTS   AGE

flask-app-598978c57-96zp9   1/1     Running   0          2m

flask-app-598978c57-zhhwg   1/1     Running   0          2m

mongodb-0                   1/1     Running   0          2m

### 4. Access the Flask App

```bash
minikube service flask-service --url
```

Keep this terminal open and use the URL to test the endpoints.

### 5. Test the Endpoints

GET request to `/`:
```bash
curl http://127.0.0.1:<PORT>
```
Expected response:
Welcome to the Flask app! The current time is: 2026-06-15 16:00:27.971070

POST request to `/data`:
```bash
curl -X POST -H "Content-Type: application/json" -d '{"name":"satyam","role":"developer"}' http://127.0.0.1:<PORT>/data
```
Expected response:
```json
{"status": "Data inserted"}
```

GET request to `/data`:
```bash
curl http://127.0.0.1:<PORT>/data
```
Expected response:
```json
[{"name": "satyam", "role": "developer"}]
```

---

## DNS Resolution in Kubernetes

In Kubernetes, every Service gets a DNS name automatically assigned by the internal DNS system (CoreDNS). This allows pods to communicate with each other using service names instead of IP addresses.

When the Flask app needs to connect to MongoDB, it does not use an IP address. Instead it uses the service name `mongodb` directly in the connection string:
mongodb://admin:admin123@mongodb:27017/flask_db?authSource=admin

Kubernetes resolves `mongodb` to the ClusterIP of the MongoDB service internally. The full DNS format inside the cluster is:
<service-name>.<namespace>.svc.cluster.local

So `mongodb` resolves to `mongodb.default.svc.cluster.local` automatically. This makes inter-pod communication simple, reliable, and independent of IP addresses which can change when pods restart.

---

## Resource Requests and Limits

Resource requests and limits are configured for both Flask and MongoDB pods to ensure stable and efficient resource utilization.

- **Request**: The minimum amount of CPU/memory guaranteed to the container. Kubernetes uses this to decide which node to schedule the pod on.
- **Limit**: The maximum amount of CPU/memory the container is allowed to use. If it exceeds this, it gets throttled (CPU) or killed (memory).

Both Flask and MongoDB are configured with:

```yaml
resources:
  requests:
    cpu: "200m"
    memory: "250Mi"
  limits:
    cpu: "500m"
    memory: "500Mi"
```

This ensures no single pod can starve other pods of resources, and the cluster remains stable under load.

---

## Design Choices

### Why StatefulSet for MongoDB?
MongoDB requires stable network identity and persistent storage. StatefulSet guarantees ordered pod naming (`mongodb-0`) and stable DNS, which is critical for databases. A regular Deployment does not provide this.

### Why ClusterIP for MongoDB Service?
MongoDB should not be accessible from outside the cluster. ClusterIP exposes the service only within the cluster, so only the Flask app can reach it. This is a security best practice for databases.

### Why NodePort for Flask Service?
NodePort exposes the Flask app on a specific port on the Minikube node, making it accessible from the local machine for testing. In production, a LoadBalancer or Ingress would be used instead.

### Why Persistent Volume for MongoDB?
Without a PV and PVC, MongoDB data would be lost every time the pod restarts. The PV stores data on the host machine at `/data/mongo`, and the PVC binds to it, ensuring data persists across pod restarts.

### Why Kubernetes Secrets for MongoDB Credentials?
Hardcoding credentials in YAML files is a security risk. Kubernetes Secrets store sensitive data in base64 encoded form and inject them as environment variables at runtime, keeping credentials out of source code.

---

## Autoscaling Setup and Test Results

### HPA Configuration
- Metric: CPU utilization
- Threshold: 70%
- Minimum replicas: 2
- Maximum replicas: 5

### Enable Metrics Server

```bash
minikube addons enable metrics-server
```

### Load Test

To simulate high traffic and trigger autoscaling:

```bash
kubectl run -i --tty load-generator --image=busybox --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://flask-service:5000 & done"
```

Watch HPA in another terminal:

```bash
kubectl get hpa -w
```

### Results

| Time | CPU Usage | Replicas |
|------|-----------|----------|
| 0 min | 0% | 2 |
| 5 min | 27% | 2 |
| 8 min | 41% | 2 |
| 10 min | 87% | 3 |
| 12 min | 87% | 4 |
| 15 min | 43% | 4 |
| 20 min | 1% | 4 |
| 25 min | 1% | 2 (scaled down) |

When CPU exceeded 70%, HPA automatically scaled Flask replicas from 2 to 4. After load stopped and CPU dropped to 1%, HPA scaled back down to 2 replicas after the cooldown period.

### Screenshots

**HPA Scaling Up and Down:**

![HPA Autoscaling Results](screenshots/hpa-autoscaling.png)

**Pods Scaled to 4 Replicas:**

![Pods Scaled](screenshots/pods-scaled.png)

---

## Cookie Point: Virtual Environment Benefits

A Python virtual environment isolates project dependencies from the system Python installation. Each project gets its own set of packages and versions, so there are no conflicts between projects. For example, one project can use Flask 2.0.2 while another uses Flask 3.0 on the same machine without any issues. It also makes the project reproducible — anyone can clone the repo, create a virtual environment, and run `pip install -r requirements.txt` to get the exact same setup.

---

## Cookie Point: Testing Scenarios

### Database Interaction Testing

After deploying all resources, the MongoDB connection and data persistence were tested using the `/data` endpoint.

**POST request — inserting data:**
```bash
Invoke-RestMethod -Uri http://127.0.0.1:50899/data -Method POST -ContentType "application/json" -Body '{"name":"satyam","role":"developer"}'
```
Result:
status
Data inserted

**GET request — retrieving data:**
```bash
Invoke-RestMethod -Uri http://127.0.0.1:50899/data -Method GET
```
Result:
name   role
satyam developer

Data was successfully stored in MongoDB with authentication and retrieved correctly. The Flask app connected to MongoDB using the service DNS name `mongodb` internally.

### Autoscaling Test

A load generator pod was used to simulate high traffic:
```bash
kubectl run -i --tty load-generator --image=busybox --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://flask-service:5000 & done"
```

| Time | CPU Usage | Replicas |
|------|-----------|----------|
| 0 min | 0% | 2 |
| 5 min | 27% | 2 |
| 8 min | 41% | 2 |
| 10 min | 87% | 3 |
| 12 min | 87% | 4 |
| 20 min | 1% | 2 (scaled down) |

### Issues Encountered

| Issue | Solution |
|-------|----------|
| Docker Desktop not running when starting Minikube | Started Docker Desktop first, then ran `minikube start` |
| `docker login` access denied error | Ran VS Code as Administrator |
| HPA showing `<unknown>` CPU | Enabled metrics-server addon and waited 1-2 minutes |
| CPU not crossing 70% with single load generator | Used aggressive background load with `& done` to spawn parallel requests |

### Screenshots

**HPA Autoscaling — Scale Up and Scale Down:**

![HPA Autoscaling Results](screenshots/hpa-autoscaling.png)

**Pods Scaled to 4 Replicas Under Load:**

![Pods Scaled](screenshots/pods-scaled.png)

---

## Cleanup

To delete all resources:

```bash
kubectl delete -f k8s/
minikube stop
```