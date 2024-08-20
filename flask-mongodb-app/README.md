# Flask MongoDB Kubernetes Deployment

## Prerequisites

- Python 3.8 or later
- Docker
- Kubernetes (Minikube, Docker Desktop, or other local Kubernetes alternatives)
- kubectl

## Setup Instructions

### Step 1: Build and Push Docker Image

1. **Build the Docker image**:

   ```bash
   docker build -t dhrypt/flask-mongodb-app .
   ```

2. **Login in to Docker Hub**:

   ```bash
   docker login
   ```

3. **Push the image to Docker Hub**:

   ```bash
   docker push dhrypt/flask-mongodb-app

   ```

### Step 2: Apply Kubernetes Confifurations

1. **Apply the Flask deployment and service**:

   ```bash
   kubectl apply -f flask-deployment.yaml
   kubectl apply -f flask-service.yaml
   ```

2. **Apply the MongoDB StatefulSet and service**:

   ```bash
   kubectl apply -f mongodb-statefulset.yaml
   kubectl apply -f mongodb-service.yaml
   ```

3. **Apply the Horizontal Pod Autoscaler**:

   ```bash
   kubectl apply -f flask-hpa.yaml
   ```

### Step 3: Access the Application

1. **Check the status of the services**:

   ```bash
   kubectl get services
   ```

2. Access the application at `http://localhost:5000`.

### Step 4: Test Endpoints

1. **Test the root endpoint**:

   ```bash
   curl http://localhost:5000/
   ```

2. **Test the `/data` endpoint with a POST request**:

   ```bash
   curl -X POST -H "Content-Type: application/json" -d '{"sampleKey":"sampleValue"}' http://localhost:5000/data
   ```

3. **Test the `/data` endpoint with a GET request**:

   ```bash
   curl http://localhost:5000/data
   ```

4. **Autoscaling Test**:

To simulate high traffic, we used Apache Benchmark to send a large number of requests to the Flask application and monitored the HPA to ensure the number of replicas increased as CPU usage exceeded 70%.

```bash
(venv) ➜ flask-mongodb-app ab -n 1000 -c 100 http://localhost:5000/


 This is ApacheBench, Version 2.3 <$Revision: 1913912 $>
 Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
 Licensed to The Apache Software Foundation, http://www.apache.org/

 Benchmarking localhost (be patient)
 Completed 100 requests
 Completed 200 requests
 Completed 300 requests
 Completed 400 requests
 Completed 500 requests
 Completed 600 requests
 Completed 700 requests
 Completed 800 requests
 Completed 900 requests
 Completed 1000 requests
 Finished 1000 requests

 Server Software: Werkzeug/2.0.3
 Server Hostname: localhost
 Server Port: 5000

 Document Path: /
 Document Length: 73 bytes

 Concurrency Level: 100
 Time taken for tests: 0.794 seconds
 Complete requests: 1000
 Failed requests: 0
 Total transferred: 227000 bytes
 HTML transferred: 73000 bytes
 Requests per second: 1259.71 [#/sec] (mean)
 Time per request: 79.383 [ms] (mean)
 Time per request: 0.794 [ms] (mean, across all concurrent requests)
 Transfer rate: 279.25 [Kbytes/sec] received

 Connection Times (ms)
 min mean[+/-sd] median max
 Connect: 0 0 0.7 0 4
 Processing: 5 71 26.2 69 154
 Waiting: 5 70 25.8 68 151
 Total: 9 72 25.8 69 154

 Percentage of the requests served within a certain time (ms)
 50% 69
 66% 79
 75% 88
 80% 90
 90% 106
 95% 121
 98% 135
 99% 149
 100% 154 (longest request)
```

**Results:**

- Initial number of replicas: 2
- Number of replicas after traffic: 4
- CPU usage: Exceeded 70%, triggering the HPA to scale the number of replicas.

No issues were encountered during the testing.

## Detailed Kubernetes YAML Files

**flask-deployment.yaml**

Defines the deployment for the Flask application, ensuring 2 replicas for high availability. It also includes resource requests and limits.

**flask-service.yaml**

Creates a LoadBalancer service to expose the Flask application externally.

**mongodb-statefulset.yaml**

Configures MongoDB StatefulSet with authentication enabled and persistent storage using PVC.

**mongodb-service.yaml**

Creates a ClusterIP service for MongoDB, making it accessible only within the cluster.

**flask-hpa.yaml**

Sets up Horizontal Pod Autoscaler (HPA) to scale the Flask deployment based on CPU usage.

## Explanation of DNS Resolution

In a Kubernetes cluster, DNS resolution allows pods to communicate with each other using service names. Each service is assigned a DNS name, which is managed by the internal DNS service provided by Kubernetes. For example, the Flask application can connect to MongoDB using the service name `mongodb`, which resolves to the IP address of the MongoDB pod.

**Example:**

```python
# Flask app connecting to MongoDB using service name
client = MongoClient(f"mongodb://{mongo_username}:{mongo_password}@mongodb:27017/")
```

## Resource Requests and Limits

Resource requests and limits ensure efficient utilization of cluster resources. Requests guarantee a certain amount of CPU and memory for the pod, while limits set a maximum cap.

**Example:**

In `flask-deployment.yaml`:

```yaml
resources:
  requests:
    memory: "250Mi"
    cpu: "200m"
  limits:
    memory: "500Mi"
    cpu: "500m"
```

This configuration ensures that each Flask pod requests 250Mi of memory and 200m of CPU, and it cannot exceed 500Mi of memory and 500m of CPU.

## Design Choices

### Flask Application

We chose Flask for its simplicity and efficiency in creating RESTful APIs. It is lightweight, easy to set up, and well-suited for small to medium-sized applications like ours. Flask's modular design and extensive documentation make it an ideal choice for rapid development.

### MongoDB

MongoDB was selected for its flexibility in handling JSON-like documents, which aligns perfectly with the structure of data managed by our Flask application. Its scalability and native support for sharding and replication make it an excellent choice for applications expected to grow in data volume and complexity.

### Kubernetes

Kubernetes was chosen for its robust orchestration capabilities, allowing us to manage containerized applications efficiently. It provides essential features like automatic scaling, self-healing, and rolling updates, ensuring our application remains highly available and resilient.

### Docker Desktop Kubernetes

Docker Desktop Kubernetes was used to provide a local Kubernetes environment. This setup is ideal for development and testing purposes, allowing us to simulate a real Kubernetes cluster on a local machine without the need for additional tools like Minikube.

### Horizontal Pod Autoscaler (HPA)

The HPA was configured to automatically scale the Flask application based on CPU usage. This ensures that the application can handle varying loads efficiently, maintaining performance during high traffic periods and scaling down during low usage to save resources.

### Persistent Volume and Persistent Volume Claim

Using a PV and PVC for MongoDB ensures data persistence, even if the MongoDB pod is restarted or rescheduled. This setup guarantees that our database remains consistent and available, providing a reliable backend for our application.

### Secrets

Kubernetes Secrets were used to store MongoDB credentials securely, following best practices for sensitive data management. By using Secrets, we ensure that sensitive information such as database usernames and passwords are not exposed in plain text within our configuration files.

## Deployment Choices

- **Flask Deployment**: Implemented with 2 replicas for high availability. Resource requests and limits were set to ensure efficient resource utilization and stability.
  MongoDB StatefulSet: Ensures stable, unique network identifiers and persistent storage, crucial for database operations.

- **Services**:
  Flask Service: Created as a LoadBalancer to expose the Flask application externally.
  MongoDB Service: Configured as a ClusterIP service to restrict MongoDB access to within the cluster, enhancing security.

## Alternatives Considered

- **Single Replica for Flask**: Not chosen due to lack of redundancy and high availability.
- **Deployment for MongoDB**: Lacks stable network identifiers and persistent storage provided by StatefulSets.
- **NodePort for Flask Service**: Potential port conflicts and less flexibility compared to LoadBalancer.
- **Ephemeral Storage for MongoDB**: High risk of data loss if the pod is deleted or rescheduled.
- **Manual Scaling for Flask**: Lacks responsiveness to real-time traffic changes, unlike HPA.
