# Node.js Docker Kubernetes Setup Guide (Windows)

## Project Structure

```
nodejs-k8s-app/
├── app/
│   ├── node_modules/        (created after npm install)
│   ├── package.json
│   ├── package-lock.json    (created after npm install)
│   ├── server.js
│   └── public/
│       └── index.html
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
├── Dockerfile
├── .dockerignore
└── package-lock.json        (optional, at root level)
```

**Important**: Your `package.json` is inside the `app/` folder, so all npm commands must be run from there.

## Prerequisites Installation (Windows)

### 1. Install Node.js
- Download from: https://nodejs.org/
- Choose LTS version
- Verify installation:
```powershell
node --version
npm --version
```

### 2. Install Docker Desktop
- Download from: https://www.docker.com/products/docker-desktop
- Install and restart your computer
- Enable WSL 2 backend if prompted
- Verify installation:
```powershell
docker --version
docker ps
```

### 3. Install Minikube
```powershell
# Using Chocolatey (install Chocolatey first if not installed)
choco install minikube

# OR download installer from:
# https://minikube.sigs.k8s.io/docs/start/

# Verify installation
minikube version
```

### 4. Install kubectl
```powershell
# Using Chocolatey
choco install kubernetes-cli

# OR download from:
# https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/

# Verify installation
kubectl version --client
```

---

## Step-by-Step Deployment

### Step 1: Run Application Locally

1. Create project structure and add all files
2. Navigate to the **app** directory (this is important!):
```powershell
cd nodejs-k8s-app/app
```

3. Install dependencies:
```powershell
npm install
```

4. Run the application:
```powershell
npm start
```

5. Open browser and visit: `http://localhost:3000`

**Note**: All npm commands must be run from inside the `app/` folder since that's where `package.json` is located.

---

### Step 2: Docker Deployment with Bridge Network

#### Build Docker Image
```powershell
# Navigate to the ROOT of the project (where Dockerfile is)
cd nodejs-k8s-app

# Build the image
docker build -t nodejs-k8s-app:latest .
```

**Important**: The Docker build command must be run from the root `nodejs-k8s-app/` folder (where the Dockerfile is), NOT from the `app/` folder.

#### Create Custom Bridge Network
```powershell
# Create a custom bridge network
docker network create --driver bridge my-app-network

# Inspect the network
docker network inspect my-app-network
```

#### Run Container with Default Bridge Network
```powershell
# Run on default bridge network
docker run -d -p 3000:3000 --name nodejs-app nodejs-k8s-app:latest

# Access the application
# Visit: http://localhost:3000

# Check logs
docker logs nodejs-app

# Stop and remove
docker stop nodejs-app
docker rm nodejs-app
```

#### Run Container with Custom Bridge Network
```powershell
# Run on custom bridge network
docker run -d -p 3000:3000 --name nodejs-app --network my-app-network nodejs-k8s-app:latest

# Inspect container network
docker inspect nodejs-app | Select-String "NetworkMode|IPAddress"

# Access the application
# Visit: http://localhost:3000
```

#### Run Multiple Containers with Inter-Container Communication
```powershell
# Run first instance
docker run -d -p 3001:3000 --name nodejs-app-1 --network my-app-network nodejs-k8s-app:latest

# Run second instance
docker run -d -p 3002:3000 --name nodejs-app-2 --network my-app-network nodejs-k8s-app:latest

# Test communication between containers
docker exec nodejs-app-1 ping nodejs-app-2

# Access applications
# Visit: http://localhost:3001
# Visit: http://localhost:3002
```

#### Docker Network Commands
```powershell
# List all networks
docker network ls

# Inspect a network
docker network inspect my-app-network

# Connect a running container to a network
docker network connect my-app-network container-name

# Disconnect a container from a network
docker network disconnect my-app-network container-name

# Remove a network
docker network rm my-app-network
```

#### Cleanup Docker
```powershell
# Stop all containers
docker stop nodejs-app-1 nodejs-app-2

# Remove containers
docker rm nodejs-app-1 nodejs-app-2

# Remove network
docker network rm my-app-network
```

---

### Step 3: Minikube Deployment with Kubernetes Networking

#### Start Minikube
```powershell
# Start Minikube with Docker driver
minikube start --driver=docker

# Verify Minikube is running
minikube status

# Enable addons (optional)
minikube addons enable metrics-server
minikube addons enable dashboard
```

#### Build Image in Minikube's Docker Environment
```powershell
# Point shell to Minikube's Docker daemon
& minikube -p minikube docker-env --shell powershell | Invoke-Expression

# Build the image
docker build -t nodejs-k8s-app:latest .

# Verify image exists in Minikube
docker images | Select-String nodejs-k8s-app
```

#### Deploy to Kubernetes
```powershell
# Apply deployment
kubectl apply -f k8s/deployment.yaml

# Apply service
kubectl apply -f k8s/service.yaml

# Check deployment status
kubectl get deployments
kubectl get pods
kubectl get services

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app=nodejs-app --timeout=60s
```

#### Access the Application
```powershell
# Get Minikube IP
minikube ip

# Get service URL
minikube service nodejs-app-service --url

# OR open in browser automatically
minikube service nodejs-app-service
```

The application will be accessible at: `http://<minikube-ip>:30080`

#### Kubernetes Networking Concepts Demonstrated

**1. ClusterIP (Internal Communication)**
- Pods communicate internally using DNS: `nodejs-app-service.default.svc.cluster.local`
- Each pod gets its own IP address
- Service provides load balancing across pods

**2. NodePort (External Access)**
- Service exposed on port 30080 on each node
- Traffic flow: External → NodePort (30080) → Service (3000) → Pod (3000)

**3. Pod-to-Pod Communication**
```powershell
# Get pod names
kubectl get pods

# Execute into a pod
kubectl exec -it <pod-name> -- sh

# Inside pod, test communication
wget -qO- http://nodejs-app-service:3000/health
```

#### Kubernetes Commands
```powershell
# View pod details
kubectl describe pod <pod-name>

# View service details
kubectl describe service nodejs-app-service

# View logs
kubectl logs <pod-name>

# Scale deployment
kubectl scale deployment nodejs-app-deployment --replicas=5

# Check events
kubectl get events --sort-by=.metadata.creationTimestamp

# Port forward to local machine
kubectl port-forward service/nodejs-app-service 8080:3000
# Access at: http://localhost:8080
```

#### Access Kubernetes Dashboard
```powershell
# Start dashboard
minikube dashboard
```

---

## Network Architecture Overview

### Docker Bridge Network
```
Host Machine (Windows)
    ├── Docker Bridge Network (172.17.0.0/16)
    │   ├── Container 1 (172.17.0.2)
    │   └── Container 2 (172.17.0.3)
    └── Custom Bridge Network (my-app-network)
        ├── Container 1 (can communicate by name)
        └── Container 2 (can communicate by name)
```

### Kubernetes (Minikube) Network
```
Minikube VM/Container
    ├── Pod Network (CNI - Container Network Interface)
    │   ├── Pod 1 (10.244.0.x)
    │   ├── Pod 2 (10.244.0.y)
    │   └── Pod 3 (10.244.0.z)
    ├── Service Network (ClusterIP)
    │   └── nodejs-app-service (10.96.x.y:3000)
    └── NodePort
        └── Host:30080 → Service:3000 → Pod:3000
```

---

## Testing and Verification

### Test Docker Networking
```powershell
# Check container network settings
docker inspect nodejs-app | Select-String "Networks" -Context 10

# Test connectivity
docker exec nodejs-app curl http://localhost:3000/health
```

### Test Kubernetes Networking
```powershell
# Test service endpoint
kubectl get endpoints nodejs-app-service

# Test from within cluster
kubectl run test-pod --image=busybox -it --rm -- wget -qO- http://nodejs-app-service:3000/health

# Test DNS resolution
kubectl run test-pod --image=busybox -it --rm -- nslookup nodejs-app-service
```

---

## Cleanup

### Docker Cleanup
```powershell
# Stop and remove containers
docker stop $(docker ps -aq)
docker rm $(docker ps -aq)

# Remove image
docker rmi nodejs-k8s-app:latest

# Remove networks
docker network prune
```

### Kubernetes Cleanup
```powershell
# Delete resources
kubectl delete -f k8s/service.yaml
kubectl delete -f k8s/deployment.yaml

# OR delete by label
kubectl delete all -l app=nodejs-app

# Stop Minikube
minikube stop

# Delete Minikube cluster
minikube delete
```

---

## Troubleshooting

### Docker Issues
```powershell
# Check Docker daemon
docker info

# Check container logs
docker logs nodejs-app

# Check container processes
docker top nodejs-app
```

### Kubernetes Issues
```powershell
# Check pod status
kubectl get pods -o wide

# Describe pod for events
kubectl describe pod <pod-name>

# Check logs
kubectl logs <pod-name>

# Check service endpoints
kubectl get endpoints

# Check Minikube logs
minikube logs
```

### Common Issues

**Port Already in Use:**
```powershell
# Find process using port 3000
netstat -ano | findstr :3000
# Kill the process
taskkill /PID <process-id> /F
```

**Minikube Won't Start:**
```powershell
# Delete and restart
minikube delete
minikube start --driver=docker
```

**Image Not Found in Minikube:**
```powershell
# Make sure to build in Minikube's Docker environment
& minikube docker-env --shell powershell | Invoke-Expression
docker build -t nodejs-k8s-app:latest .
```

---

## Additional Docker Network Modes

### Host Network Mode
```powershell
# Container uses host's network stack directly
# Note: Not available on Docker Desktop for Windows
# Only works on Linux Docker hosts
docker run -d --network host --name nodejs-app nodejs-k8s-app:latest
```

### None Network Mode
```powershell
# Container has no network access
docker run -d --network none --name nodejs-app nodejs-k8s-app:latest
```

---

## Monitoring and Observability

### Docker Stats
```powershell
# View container resource usage
docker stats nodejs-app
```

### Kubernetes Monitoring
```powershell
# View pod resource usage
kubectl top pods

# View node resource usage
kubectl top nodes

# Stream logs
kubectl logs -f <pod-name>
```

---

## Next Steps

1. **Add Ingress Controller**: For HTTP/HTTPS routing
2. **Implement ConfigMaps**: For configuration management
3. **Add Secrets**: For sensitive data
4. **Set up Persistent Storage**: For data persistence
5. **Implement CI/CD Pipeline**: For automated deployments
6. **Add Monitoring**: Prometheus and Grafana
7. **Implement Service Mesh**: Istio for advanced networking

---

## References

- Docker Networking: https://docs.docker.com/network/
- Kubernetes Networking: https://kubernetes.io/docs/concepts/services-networking/
- Minikube Documentation: https://minikube.sigs.k8s.io/docs/