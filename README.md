# Scale Deployments Based on HTTP Requests in GKE Cluster

This guide will walk you through configuring Horizontal Pod Autoscaling (HPA) in a Google Kubernetes Engine (GKE) cluster, scaling your deployment based on HTTP requests per second (RPS).

---

### **Step 1: Create a Proxy-Only Subnetwork**
1. **Navigate to the VPC Network section in the GCP Console**.
   - Go to **VPC Network > VPC networks**.

2. **Create a new subnetwork** in the region where your GKE cluster resides (e.g., `us-central1`).
   - **Purpose**: Select **Private Google Access**.
   - **IP CIDR Range**: Choose an appropriate unused CIDR range (e.g., `10.0.0.0/24`).
   - **Region**: Select the region where your GKE cluster is located (e.g., `us-central1`).

This ensures your cluster can communicate with Google-managed services.

---

### **Step 2: Enable Gateway API on the Cluster**

Use the following command to enable Gateway API on your GKE cluster:

```bash
gcloud container clusters update CLUSTER_NAME \
    --gateway-api=standard \
    --region=COMPUTE_REGION
```
- Replace `CLUSTER_NAME` with your cluster's name.
- Replace `COMPUTE_REGION` with your cluster's region (e.g., `us-central1`).

---

### **Step 3: Create a Deployment and Service**

Create a deployment for your application and add the `networking.gke.io/max-rate-per-endpoint` annotation to the service manifest file to limit the RPS per pod.

#### **Deployment & Service Manifest:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  annotations:
    networking.gke.io/max-rate-per-endpoint: "10"    # 10 Requests per Pod per second
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

This defines a limit of **10 HTTP requests per pod per second**.

---

### **Step 4: Create a Gateway Resource**

The Gateway resource will forward HTTP traffic to the service using an HTTPRoute.

#### **Gateway Manifest:**
```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: hpa-gateway
spec:
  gatewayClassName: gke-l7-rilb
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```

---

### **Step 5: Create an HTTPRoute Resource**

Define an HTTPRoute to send traffic from the Gateway to the NGINX service.

#### **HTTPRoute Manifest:**
```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: hpa-httproute
spec:
  parentRefs:
  - kind: Gateway
    name: hpa-gateway
  rules:
  - backendRefs:
      - name: nginx-service
        port: 80
    matches:
     - path:
        type: PathPrefix
        value: /
```

---

### **Step 6: Create a HealthCheckPolicy**

Create a `HealthCheckPolicy` to monitor the health of your service and ensure proper load balancing.

#### **HealthCheckPolicy Manifest:**
```yaml
apiVersion: networking.gke.io/v1
kind: HealthCheckPolicy
metadata:
  name: nginx-healthcheck
spec:
  default:
    checkIntervalSec: 15
    timeoutSec: 15
    healthyThreshold: 3
    unhealthyThreshold: 2
    logConfig:
      enabled: true
    config:
      type: HTTP
      httpHealthCheck:
        portSpecification: USE_FIXED_PORT
        port: 80
        requestPath: /
  targetRef:
    group: ""
    kind: Service
    name: nginx-service
```

This checks the health of the NGINX service every 15 seconds.

---

### **Step 7: Configure HPA for HTTP-Based Scaling**

Configure the Horizontal Pod Autoscaler (HPA) to scale the deployment based on the custom metric `autoscaling.googleapis.com|gclb-capacity-utilization`, which measures load balancer capacity utilization.

#### **HPA Manifest:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Object
    object:
      describedObject:
        kind: Service
        name: nginx-service
      metric:
        name: "autoscaling.googleapis.com|gclb-capacity-utilization"
      target:
        averageValue: 60
        type: AverageValue
```

### **Explanation**:
- **`max-rate-per-endpoint: "10"`**: Sets a limit of 10 requests per pod per second in the NGINX service.
- **`autoscaling.googleapis.com|gclb-capacity-utilization`**: Custom metric that represents the percentage of traffic capacity used by each pod.
- **`averageValue: 60`**: This indicates the target for scaling, meaning the deployment will scale when **60% of the maximum RPS** (i.e., 6 requests per pod) is reached.

As load increases, the HPA will scale up pods, and as the load decreases, it will scale them down accordingly.

---

Following this guide, you have now successfully set up your GKE cluster to automatically scale your NGINX deployment based on the number of HTTP requests per second.
