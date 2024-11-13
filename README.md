# Scale Deployments Based on HTTP Requests in GKE Cluster

This guide will walk you through configuring Horizontal Pod Autoscaling (HPA) in a Google Kubernetes Engine (GKE) cluster, scaling your deployment based on HTTP requests per second (RPS).

---

### **Step 1: Create a Proxy-Only Subnetwork**
1. **Navigate to the VPC Network section in the GCP Console**.
   - Go to **VPC Network > VPC networks > Subnets > Add Subnet**.

#### **Configure the Subnetwork Properties**

Fill out the necessary fields to create the Proxy-Only subnetwork:

1. **Name**: Provide a unique name for your subnetwork (e.g., `proxy-only-subnet`).

2. **Region**: Select the region where your **Regional Internal Load Balancer (RILB)** will be deployed. This is often `us-central1`, but confirm this based on your architecture needs.

3. **IP CIDR Range**: Choose a suitable, unused **CIDR range** for the subnetwork. This range should not overlap with other networks in your VPC (e.g., `10.0.2.0/24`).

4. **Purpose**: Choose **Regional Managed Proxy**. This setting ensures that only private Google services (such as load balancing) can access this subnetwork.

5. **Role**: Ensure this option is **Active**.

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
      type: TCP
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



# Testing the Auto-Scaling

To verify that the auto-scaling mechanism works based on HTTP requests per second, we'll use **Ddosify**, a tool to simulate high traffic on your NGINX service.



---

### **Step 1: Get the Gateway IP Address**
First, retrieve the IP address of the Gateway created earlier to route traffic to the NGINX service:

```bash
kubectl get gateway
```
- This command will display the IP of the Gateway. **Copy** the IP address from the output.

---

### **Step 2: Run a Ddosify Pod**
Now, create a pod running **Ddosify** to simulate the HTTP requests load:

```bash
kubectl run ddosify --image=ddosify/ddosify --command -- sleep 7200
```
- This creates a pod with the Ddosify image and keeps it running for 2 hours (7200 seconds), allowing you to perform multiple tests.

---

### **Step 3: Exec Into the Ddosify Pod**
Once the pod is running, access the Ddosify pod's shell to run your traffic test:

```bash
kubectl exec -it ddosify -- /bin/sh
```
- This opens an interactive shell inside the running Ddosify pod.

---

### **Step 4: Run the Traffic Test**
Simulate **2000 HTTP requests in 20 seconds** to test the auto-scaling:

```bash
ddosify -n 20000 -d 20 -t http://GATEWAY_IP
```
- Replace `GATEWAY_IP` with the IP address of the Gateway from **Step 1**.
- **-n 2000**: This means 2000 requests will be sent.
- **-d 20**: The duration of the test is 20 seconds.

---

### **Expected Outcome:**
- The Horizontal Pod Autoscaler (HPA) will monitor the **requests per second** (RPS) reaching each pod.
- If the load exceeds the threshold (`6 RPS` per pod, as defined earlier), the HPA will scale up the number of replicas to handle the traffic.

After the traffic simulation, you can observe the scaling events by running:

```bash
kubectl get hpa 
```

Alternatively, describe the HPA to view scaling history and metrics:

```bash
kubectl describe hpa nginx-hpa
```
Or you can put the pods on watch mode to monitor them:

```bash
watch kubectl get pods
```

By running this test, you'll confirm that your GKE deployment scales dynamically based on HTTP traffic.

Following this guide, you have now successfully set up your GKE cluster to automatically scale your NGINX deployment based on the number of HTTP requests per second.


## Example of apply hpa(horizantal pod autoscaler) on citation deployment
      
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        labels:
          app.kubernetes.io/managed-by: Helm
        name: vertexai-citation-deployment
        namespace: default
      spec:
        replicas: 1
        selector:
          matchLabels:
            app: vertexai-citation
        strategy:
          rollingUpdate:
            maxSurge: 1
            maxUnavailable: 0
          type: RollingUpdate
        template:
          metadata:
            labels:
              app: vertexai-citation
          spec:
            containers:
            - image: gcr.io/world-learning-400909/vertexai-citation:old
              imagePullPolicy: Always
              name: vertexai-citation
              ports:
              - containerPort: 5000
                protocol: TCP
            restartPolicy: Always
            serviceAccountName: gke-sa
      
      ---
      
      apiVersion: v1
      kind: Service
      metadata:
        name: vertexai-citation-service
        namespace: default
        labels:
          app: vertexai-citation
        annotations:
          networking.gke.io/load-balancer-type: Internal
          networking.gke.io/max-rate-per-endpoint: "50"    # 50 Requests per Pod per second
      spec:
        selector:
          app: vertexai-citation
        ports:
          - protocol: TCP
            port: 80              # Port for the Service (external)
            targetPort: 5000       # Port exposed by the container
        type: LoadBalancer         # Exposes the Service with an external IP
      
      
      ---
      
      apiVersion: gateway.networking.k8s.io/v1beta1
      kind: Gateway
      metadata:
        name: hpa-gateway-citation
      spec:
        gatewayClassName: gke-l7-rilb
        listeners:
          - name: http
            protocol: HTTP
            port: 80
      
      ---
      
      apiVersion: gateway.networking.k8s.io/v1beta1
      kind: HTTPRoute
      metadata:
        name: hpa-httproute
      spec:
        parentRefs:
        - kind: Gateway
          name: hpa-gateway-citation
        rules:
        - backendRefs:
            - name: vertexai-citation-service
              port: 80
          matches:
           - path:
              type: PathPrefix
              value: /
      
      ---
      
      apiVersion: networking.gke.io/v1
      kind: HealthCheckPolicy
      metadata:
        name: vertexai-citation-healthcheck
      spec:
        default:
          checkIntervalSec: 15
          timeoutSec: 15
          healthyThreshold: 3
          unhealthyThreshold: 2
          logConfig:
            enabled: true
          config:
            type: TCP
            httpHealthCheck:
              portSpecification: USE_FIXED_PORT
              port: 80
              requestPath: /
        targetRef:
          group: ""
          kind: Service
          name: vertexai-citation-service
      
      ---
      
      apiVersion: autoscaling/v2
      kind: HorizontalPodAutoscaler
      metadata:
        name: vertexai-citation-hpa
      spec:
        scaleTargetRef:
          apiVersion: apps/v1
          kind: Deployment
          name: vertexai-citation-deployment
        minReplicas: 1
        maxReplicas: 10
        metrics:
        - type: Object
          object:
            describedObject:
              kind: Service
              name: vertexai-citation-service
            metric:
              name: "autoscaling.googleapis.com|gclb-capacity-utilization"
            target:
              averageValue: 60
              type: AverageValue


      remember: if your have multiple deployment and you need to apply **hpa** on all deployment, so you must create saparate gateway , httproute, healthcheckpolicy and hpa for each deployment service, like i have created all the resources for citation deployment. Other wise it get failed. you can also create saparate resources using same configuration by giving them unique resources names. it will create saparate resource for the the deployment and works fine.

      2nd is testing should be done on the test enviroment, but when you work on any client instance, make sure not to delete the existing resources, try to find the way to fix the issue, like i have delete deployment and service, instead i got deployment and service name and use these name with my httproute , healthcheckpolicy and on hpa. with this approach, everything is working perfectly fine, and i have not deleted any thing..

## Error what i faced during the hpa deployment

   i deployed all the thing, but when i sent traffic on gateway using curl it get failed

   kubectl get gateway     ---> get gateway ip from here,

   now go inside any pod and install curl in it, and curl the gateway,

   curl http://gateway-ip    ---> i got no response

   step for resolving the issue:
   ----------------------------

   1- i first check service and deployment which are working fine or not, by sending traffic to service.

   i get service ip with command "k get svc" and went inside the any pod and sent traffic with curl using service ip to the service, it work fine, i got idea that my service and deployment is working fine..

   2- next i describe all resource like gateway, httproute and healthcheckpolicy, hpa, everything give me response like ""success"", but my gateway could not send traffic to service.. i did some chatgpt and had found i need to check GCP load balancer, i went to the gcp load balancer and see that my gateway ip is same a load balancer ip, and my loadbalacer gave me same "healthcheck issue" like showing some unhealthy backend, 

For solution:
------------
 
      I just changed healthcheckpolicy protocal from HTTP to TCP and it works fine, because healthcheck was applied on service and service was using the protocal of TCP not  HTTP, that why my traffic was not sent to service through gateway, after that it works fine. i test this by sending multiple http request traffic to the deployment and HPA scale the pods accordingly


