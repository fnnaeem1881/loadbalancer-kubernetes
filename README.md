
# Node.js Kubernetes Load Balancer Project - Complete Documentation

This guide provides a step-by-step walkthrough for creating, deploying, and testing a Node.js application on Kubernetes with a Load Balancer.

## 📌 Table of Contents
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Step 1: Create Node.js Application](#step-1-create-nodejs-application)
- [Step 2: Dockerize the Application](#step-2-dockerize-the-application)
- [Step 3: Set Up Kubernetes Deployment & Service](#step-3-set-up-kubernetes-deployment--service)
- [Step 4: Deploy to Kubernetes](#step-4-deploy-to-kubernetes)
- [Step 5: Test Load Balancing](#step-5-test-load-balancing)
- [Step 6: Scaling & Monitoring](#step-6-scaling--monitoring)
- [Clean Up](#clean-up)
- [Troubleshooting](#troubleshooting)
- [Conclusion](#conclusion)

---

## ✅ Prerequisites
Before starting, ensure you have:
- [Docker installed](https://docs.docker.com/get-docker/)
- [Kubernetes](https://kubernetes.io/docs/setup/) (Minikube, Docker Desktop, or cloud-based cluster)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) (Kubernetes CLI)
- [Node.js (v14+)](https://nodejs.org/) (Install Node.js)

## 📂 Project Structure
```
nodejs-k8s-loadbalancer/
├── app/
│   ├── server.js         # Node.js server
│   ├── package.json      # Node.js dependencies
│   └── Dockerfile        # Docker configuration
├── k8s/
│   ├── deployment.yaml   # Kubernetes Deployment
│   └── service.yaml      # Kubernetes Service (LoadBalancer)
└── README.md
```

## 🚀 Step 1: Create Node.js Application
1. **Initialize Node.js Project**
   ```bash
   mkdir nodejs-k8s-loadbalancer && cd nodejs-k8s-loadbalancer
   mkdir app && cd app
   npm init -y
   npm install express
   ```

2. **Create `server.js`**
   ```javascript
   const express = require('express');
   const os = require('os');

   const app = express();
   const port = process.env.PORT || 3000;

   app.get('/', (req, res) => {
     res.send(`Hello from ${os.hostname()}!`);
   });

   app.get('/health', (req, res) => {
     res.status(200).send('OK');
   });

   app.listen(port, () => {
     console.log(`Server running on port ${port}`);
   });
   ```

3. **Create `package.json`**
   ```json
   {
     "name": "nodejs-loadbalancer-demo",
     "version": "1.0.0",
     "main": "server.js",
     "scripts": {
       "start": "node server.js"
     },
     "dependencies": {
       "express": "^4.18.2"
     }
   }
   ```

## 🐳 Step 2: Dockerize the Application
1. **Create `Dockerfile`**
   ```dockerfile
   FROM node:18-alpine
   WORKDIR /usr/src/app
   COPY package*.json ./
   RUN npm install
   COPY . .
   EXPOSE 3000
   CMD ["npm", "start"]
   ```

2. **Build Docker Image**
   ```bash
   docker build -t nodejs-loadbalancer-demo:latest .
   ```

## ⚙️ Step 3: Set Up Kubernetes Deployment & Service
1. **Create `k8s/deployment.yaml`**
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nodejs-app
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: nodejs-app
     template:
       metadata:
         labels:
           app: nodejs-app
       spec:
         containers:
         - name: nodejs-app
           image: nodejs-loadbalancer-demo:latest
           imagePullPolicy: Never  # Use local image (for Minikube)
           ports:
           - containerPort: 3000
           readinessProbe:
             httpGet:
               path: /health
               port: 3000
             initialDelaySeconds: 5
             periodSeconds: 5
   ```

2. **Create `k8s/service.yaml` (LoadBalancer)**
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: nodejs-service
   spec:
     type: LoadBalancer
     selector:
       app: nodejs-app
     ports:
       - protocol: TCP
         port: 80
         targetPort: 3000
   ```

## ⚙️ Step 4: Deploy to Kubernetes
1. **Start Minikube (if using local cluster)**
   ```bash
   minikube start
   eval $(minikube docker-env)  # Use Minikube's Docker daemon
   ```

2. **Apply Kubernetes Configs**
   ```bash
   kubectl apply -f k8s/deployment.yaml
   kubectl apply -f k8s/service.yaml
   ```

3. **Check Deployment Status**
   ```bash
   kubectl get pods
   kubectl get services
   ```

4. **Enable LoadBalancer Access (Minikube)**
   ```bash
   minikube tunnel
   (Keep this running in a separate terminal.)
   ```

## 🔍 Step 5: Test Load Balancing
1. **Get Service URL**
   ```bash
   minikube service nodejs-service --url
   ```

   Example output:
   ```bash
   http://192.168.49.2:30276
   ```

2. **Send Requests to Verify Load Balancing**
   ```bash
   for i in {1..10}; do curl http://192.168.49.2:30276; done
   ```

   Expected output (different hostnames per request):
   ```bash
   Hello from nodejs-app-abc123!
   Hello from nodejs-app-xyz456!
   Hello from nodejs-app-def789!
   ...
   ```

## 📈 Step 6: Scaling & Monitoring
1. **Scale Up/Down**
   ```bash
   kubectl scale deployment nodejs-app --replicas=5
   kubectl get pods
   ```

2. **View Logs**
   ```bash
   kubectl logs <pod-name>
   ```

3. **Delete a Pod (Test Auto-Healing)**
   ```bash
   kubectl delete pod <pod-name>
   kubectl get pods  # New pod should replace the deleted one
   ```

## 🧹 Clean Up
```bash
kubectl delete -f k8s/deployment.yaml
kubectl delete -f k8s/service.yaml
minikube stop
```

## 🛠 Troubleshooting

| Issue                     | Solution                                                    |
|---------------------------|-------------------------------------------------------------|
| ImagePullBackOff           | Ensure Docker image is built (`docker build`) and correct in `deployment.yaml` |
| minikube tunnel fails      | Restart Minikube (`minikube stop && minikube start`)         |
| No load balancing          | Check replicas (`kubectl get pods`), ensure multiple pods are running |
| Connection refused         | Ensure minikube tunnel is running                           |

## 🎉 Conclusion
You’ve successfully:
✅ Created a Node.js app  
✅ Dockerized it  
✅ Deployed to Kubernetes  
✅ Tested load balancing  

Next steps:
- Add Ingress for domain routing
- Use Helm for package management
- Set up monitoring (Prometheus + Grafana)