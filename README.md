# DESC_Kubernates_Networking_Demo

# DESC_Kubernates_Service_and_Ingress_Demo

This repository contains a beginner-friendly Kubernetes/K3s demo for understanding **Services** and **Ingress** using a simple online shop scenario.

The demo covers:

- ClusterIP Service
- NodePort Service
- Ingress
- Path-based routing
- Service discovery
- Internal vs external access
- K3s Traefik Ingress Controller
- Best practices and troubleshooting

This demo works on:

- K3s
- Kubernetes
- Minikube
- kubeadm-based clusters
- EKS, AKS, and GKE

For K3s, Traefik is usually installed by default, so Ingress normally works without installing an extra Ingress Controller.

---

## Repository Files

```text
DESC_Kubernates_Service_and_Ingress_Demo/
├── 01-namespace.yaml
├── 02-products-deployment.yaml
├── 03-orders-deployment.yaml
├── 04-products-service-clusterip.yaml
├── 05-orders-service-clusterip.yaml
├── 06-products-service-nodeport.yaml
├── 07-ingress.yaml
├── 08-ingress-with-rewrite-traefik.yaml
├── 09-cleanup.sh
└── README.md
```

| File | Purpose |
|---|---|
| `01-namespace.yaml` | Creates a separate namespace for the demo |
| `02-products-deployment.yaml` | Deploys the Products application |
| `03-orders-deployment.yaml` | Deploys the Orders application |
| `04-products-service-clusterip.yaml` | Creates an internal ClusterIP Service for Products |
| `05-orders-service-clusterip.yaml` | Creates an internal ClusterIP Service for Orders |
| `06-products-service-nodeport.yaml` | Exposes Products externally using NodePort |
| `07-ingress.yaml` | Exposes Products and Orders using Ingress path routing |
| `08-ingress-with-rewrite-traefik.yaml` | Optional Traefik rewrite/strip-prefix example |
| `09-cleanup.sh` | Deletes the demo namespace and all resources |

---

# Demo Scenario

A small company called **DESC Shop** is moving its applications to Kubernetes.

The company has two applications:

1. **Products Service**
   - Shows product-related information.
   - Example real-world usage: product catalog, item details, stock list.

2. **Orders Service**
   - Shows order-related information.
   - Example real-world usage: customer orders, checkout status, delivery status.

The company wants:

- Applications to communicate internally using Services.
- Developers to test one application externally using NodePort.
- Customers to access applications using a clean URL through Ingress.

Final access design:

```text
http://shop.local/products  ---> Products Service
http://shop.local/orders    ---> Orders Service
```

---

# Demo Architecture

```text
User / Browser
      |
      | http://shop.local/products
      | http://shop.local/orders
      |
      v
Ingress Controller
      |
      | Path: /products
      | Path: /orders
      |
      v
Kubernetes Services
      |
      +--> products-service ---> Products Pods
      |
      +--> orders-service -----> Orders Pods
```

NodePort test path:

```text
User / Browser
      |
      | http://<NODE-IP>:30081
      |
      v
products-nodeport
      |
      v
Products Pods
```

---

# Concept Overview

## 1. What is a Kubernetes Service?

A Kubernetes Service gives a stable network address to a group of Pods.

Pods are temporary. They can be deleted, recreated, or moved to another node. When this happens, their IP addresses can change.

A Service solves this problem by giving users or other applications a stable name and IP.

Simple explanation:

```text
Pods = Workers
Service = Reception Desk
```

Workers can change, but the reception desk remains the same.

---

## 2. ClusterIP Service

ClusterIP is the default Service type.

It exposes the application only inside the Kubernetes cluster.

### Layman Example

ClusterIP is like an internal office room.

Only employees inside the building can access it. People outside the building cannot access it directly.

### Where It Is Used

ClusterIP is commonly used for:

- Databases
- Backend APIs
- Internal microservices
- Authentication services
- Payment services

### In This Demo

The following Services are ClusterIP:

```text
products-service
orders-service
```

They are used by Ingress to forward traffic to the correct Pods.

---

## 3. NodePort Service

NodePort exposes an application outside the cluster using:

```text
<Node-IP>:<NodePort>
```

Example:

```text
http://172.16.129.234:30081
```

### Layman Example

NodePort is like opening a door in the building wall.

Anyone who knows the building IP and door number can enter.

### Where It Is Used

NodePort is commonly used for:

- Labs
- Testing
- K3s home labs
- Internal demos
- Temporary developer access

### In This Demo

The Products app is exposed using NodePort:

```text
products-nodeport
```

This allows students to access Products directly without using Ingress.

---

## 4. Ingress

Ingress exposes HTTP and HTTPS routes from outside the cluster to Services inside the cluster.

Ingress is used when you want clean URLs such as:

```text
http://shop.local/products
http://shop.local/orders
```

Instead of using many NodePorts.

### Layman Example

Ingress is like a smart security guard at the entrance.

The guard checks where the visitor wants to go:

```text
/products -> Products Department
/orders   -> Orders Department
```

### Where It Is Used

Ingress is commonly used for:

- Websites
- APIs
- Microservices
- SaaS platforms
- Multiple applications behind one domain

### In This Demo

Ingress routes traffic like this:

| URL Path | Backend Service |
|---|---|
| `/products` | `products-service` |
| `/orders` | `orders-service` |

---

# Prerequisites

Before running this demo, make sure:

- K3s or Kubernetes is running.
- `kubectl` is installed and configured.
- You can create namespaces, deployments, services, and ingress resources.
- Ingress Controller is available.

Check cluster status:

```bash
kubectl get nodes
```

Expected example:

```text
NAME      STATUS   ROLES           AGE   VERSION
desc1     Ready    control-plane   10m   v1.xx.x+k3s1
```

Check if Traefik is running in K3s:

```bash
kubectl get pods -n kube-system | grep traefik
```

Expected example:

```text
svclb-traefik-xxxxx     Running
traefik-xxxxx           Running
```

Check IngressClass:

```bash
kubectl get ingressclass
```

Expected example:

```text
NAME      CONTROLLER
traefik   traefik.io/ingress-controller
```

---

# Step 1: Create Namespace

Apply the namespace manifest:

```bash
kubectl apply -f 01-namespace.yaml
```

Verify:

```bash
kubectl get ns
```

Expected output:

```text
service-ingress-demo   Active
```

### Command Meaning

| Command Part | Meaning |
|---|---|
| `kubectl` | Kubernetes command-line tool |
| `apply` | Creates or updates a resource |
| `-f` | Specifies the YAML file |
| `01-namespace.yaml` | Namespace manifest file |

---

# Step 2: Deploy the Products Application

Apply the Products deployment:

```bash
kubectl apply -f 02-products-deployment.yaml
```

Verify:

```bash
kubectl get pods -n service-ingress-demo
```

Expected output:

```text
products-app-xxxxxxxxx-xxxxx   1/1   Running
products-app-xxxxxxxxx-yyyyy   1/1   Running
```

### Concept

A Deployment manages Pods.

If a Products Pod is deleted, the Deployment automatically creates another one.

---

# Step 3: Deploy the Orders Application

Apply the Orders deployment:

```bash
kubectl apply -f 03-orders-deployment.yaml
```

Verify:

```bash
kubectl get pods -n service-ingress-demo
```

Expected output:

```text
orders-app-xxxxxxxxx-xxxxx   1/1   Running
orders-app-xxxxxxxxx-yyyyy   1/1   Running
```

---

# Step 4: Create ClusterIP Services

Create the Products ClusterIP Service:

```bash
kubectl apply -f 04-products-service-clusterip.yaml
```

Create the Orders ClusterIP Service:

```bash
kubectl apply -f 05-orders-service-clusterip.yaml
```

Verify Services:

```bash
kubectl get svc -n service-ingress-demo
```

Expected output:

```text
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
products-service   ClusterIP   10.x.x.x        <none>        8080/TCP
orders-service     ClusterIP   10.x.x.x        <none>        8080/TCP
```

### Important Point

ClusterIP Services are internal.

They are reachable from inside the cluster but not directly from outside.

---

# Step 5: Test ClusterIP from Inside the Cluster

Create a temporary test Pod:

```bash
kubectl run testpod -n service-ingress-demo --image=busybox -it --rm -- sh
```

Inside the test Pod, run:

```sh
wget -qO- http://products-service:8080
```

Expected output:

```text
Welcome to the Products Service
```

Test Orders Service:

```sh
wget -qO- http://orders-service:8080
```

Expected output:

```text
Welcome to the Orders Service
```

Exit the test Pod:

```sh
exit
```

### Key Point

Inside Kubernetes, applications can reach Services by name.

Example:

```text
products-service
orders-service
```

This is Kubernetes DNS.

---

# Step 6: Create NodePort Service

Apply the NodePort Service:

```bash
kubectl apply -f 06-products-service-nodeport.yaml
```

Verify:

```bash
kubectl get svc -n service-ingress-demo
```

Expected output:

```text
NAME                TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)
products-nodeport   NodePort   10.x.x.x        <none>        8080:30081/TCP
```

Get node IP:

```bash
hostname -I
```

or:

```bash
kubectl get nodes -o wide
```

Access from browser or terminal:

```bash
curl http://<NODE-IP>:30081
```

Example:

```bash
curl http://172.16.129.234:30081
```

Expected output:

```text
Welcome to the Products Service
```

### Command Meaning

| Part | Meaning |
|---|---|
| `<NODE-IP>` | IP address of your K3s node |
| `30081` | NodePort exposed on the node |
| `products-nodeport` | Service forwarding traffic to Products Pods |

---

# Step 7: Create Ingress

Apply the Ingress manifest:

```bash
kubectl apply -f 07-ingress.yaml
```

Verify:

```bash
kubectl get ingress -n service-ingress-demo
```

Expected output:

```text
NAME           CLASS     HOSTS        ADDRESS        PORTS
shop-ingress   traefik   shop.local   172.x.x.x      80
```

---

# Step 8: Configure Local DNS Using /etc/hosts

Ingress uses a hostname:

```text
shop.local
```

Your computer must know where `shop.local` points.

Get your node IP:

```bash
hostname -I
```

Edit `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Add this line:

```text
<NODE-IP> shop.local
```

Example:

```text
172.16.129.234 shop.local
```

Save and exit.

---

# Step 9: Test Ingress

Test Products path:

```bash
curl http://shop.local/products
```

Expected output:

```text
Welcome to the Products Service
```

Test Orders path:

```bash
curl http://shop.local/orders
```

Expected output:

```text
Welcome to the Orders Service
```

### What Happened?

Ingress received the request and checked the path:

```text
/products -> products-service
/orders   -> orders-service
```

Then the Service forwarded the request to the correct Pods.

---

# Optional Step 10: Ingress Rewrite / Strip Prefix

Some applications expect traffic at `/` instead of `/products` or `/orders`.

For example, an app may understand:

```text
/
```

but not:

```text
/products
```

In that case, you can use a rewrite or strip-prefix rule.

For K3s Traefik, this demo includes:

```bash
kubectl apply -f 08-ingress-with-rewrite-traefik.yaml
```

Verify:

```bash
kubectl get middleware -n service-ingress-demo
kubectl get ingress -n service-ingress-demo
```

Test again:

```bash
curl http://shop.local/products
curl http://shop.local/orders
```

---

# Full Command Sequence

```bash
# Check cluster
kubectl get nodes
kubectl get ingressclass
kubectl get pods -n kube-system | grep traefik

# Apply namespace
kubectl apply -f 01-namespace.yaml

# Deploy applications
kubectl apply -f 02-products-deployment.yaml
kubectl apply -f 03-orders-deployment.yaml

# Create ClusterIP services
kubectl apply -f 04-products-service-clusterip.yaml
kubectl apply -f 05-orders-service-clusterip.yaml

# Verify resources
kubectl get pods -n service-ingress-demo
kubectl get svc -n service-ingress-demo

# Test ClusterIP internally
kubectl run testpod -n service-ingress-demo --image=busybox -it --rm -- sh
wget -qO- http://products-service:8080
wget -qO- http://orders-service:8080
exit

# Create NodePort service
kubectl apply -f 06-products-service-nodeport.yaml
kubectl get svc -n service-ingress-demo

# Test NodePort
curl http://<NODE-IP>:30081

# Create Ingress
kubectl apply -f 07-ingress.yaml
kubectl get ingress -n service-ingress-demo

# Add to /etc/hosts
# <NODE-IP> shop.local

# Test Ingress
curl http://shop.local/products
curl http://shop.local/orders
```

---

# Best Practices

## 1. Use ClusterIP for Internal Services

Use ClusterIP for applications that should not be exposed outside the cluster.

Examples:

- Database
- Backend API
- Authentication service
- Payment service

Best practice:

```text
Database should usually be ClusterIP, not NodePort.
```

---

## 2. Use NodePort Mainly for Testing

NodePort is easy but not ideal for production.

Reasons:

- Uses high port numbers like `30081`
- Harder to manage many applications
- Less clean than domain-based access
- Can expose too much directly

Best practice:

```text
Use NodePort for labs, testing, and quick demos.
```

---

## 3. Use Ingress for HTTP/HTTPS Applications

Ingress is better when you have multiple web applications.

Examples:

```text
shop.local/products
shop.local/orders
api.company.com
portal.company.com
```

Best practice:

```text
Use Ingress for web traffic routing.
```

---

## 4. Keep Ingress and Services in the Same Namespace

Ingress rules should normally reference Services in the same namespace.

In this demo:

```text
Namespace: service-ingress-demo
Ingress: shop-ingress
Services: products-service, orders-service
```

---

## 5. Use Meaningful Labels

Services use labels to find Pods.

Example:

```yaml
selector:
  app: products-app
```

This selector must match the Pod label:

```yaml
labels:
  app: products-app
```

If labels do not match, the Service will not send traffic to any Pod.

---

## 6. Do Not Expose Databases with NodePort or Ingress

Databases should usually stay internal.

Bad practice:

```text
Internet -> Database
```

Better practice:

```text
Frontend -> Backend -> Database
```

Use ClusterIP for databases.

---

# Troubleshooting

## Problem 1: Service Does Not Respond

Check Pods:

```bash
kubectl get pods -n service-ingress-demo
```

Check Services:

```bash
kubectl get svc -n service-ingress-demo
```

Check Service endpoints:

```bash
kubectl get endpoints -n service-ingress-demo
```

If endpoints are empty, the Service selector does not match Pod labels.

Check labels:

```bash
kubectl get pods -n service-ingress-demo --show-labels
```

---

## Problem 2: NodePort Does Not Work

Check NodePort:

```bash
kubectl get svc products-nodeport -n service-ingress-demo
```

Check node IP:

```bash
kubectl get nodes -o wide
```

Try:

```bash
curl http://<NODE-IP>:30081
```

Possible causes:

| Issue | Fix |
|---|---|
| Wrong node IP | Use `kubectl get nodes -o wide` |
| Firewall blocking port | Allow port `30081` |
| Service selector mismatch | Check labels and endpoints |
| Pods not running | Check `kubectl get pods` |

---

## Problem 3: Ingress Does Not Work

Check Ingress:

```bash
kubectl get ingress -n service-ingress-demo
```

Check Traefik:

```bash
kubectl get pods -n kube-system | grep traefik
```

Check IngressClass:

```bash
kubectl get ingressclass
```

Check `/etc/hosts`:

```bash
cat /etc/hosts
```

Make sure it contains:

```text
<NODE-IP> shop.local
```

Test with curl and Host header:

```bash
curl -H "Host: shop.local" http://<NODE-IP>/products
curl -H "Host: shop.local" http://<NODE-IP>/orders
```

---

## Problem 4: 404 Not Found from Ingress

Possible causes:

| Cause | Fix |
|---|---|
| Wrong path | Use `/products` or `/orders` |
| Wrong service name in Ingress | Check `07-ingress.yaml` |
| Service has no endpoints | Check `kubectl get endpoints` |
| Ingress Controller not running | Check Traefik pods |

---

# Cleanup

Delete the complete demo:

```bash
kubectl delete namespace service-ingress-demo
```

Or run:

```bash
bash 09-cleanup.sh
```

Verify:

```bash
kubectl get ns
```

---

# Student Deliverables

Students should submit screenshots or terminal output for:

1. Nodes are ready:

```bash
kubectl get nodes
```

2. Pods are running:

```bash
kubectl get pods -n service-ingress-demo
```

3. Services are created:

```bash
kubectl get svc -n service-ingress-demo
```

4. ClusterIP internal test:

```bash
wget -qO- http://products-service:8080
wget -qO- http://orders-service:8080
```

5. NodePort test:

```bash
curl http://<NODE-IP>:30081
```

6. Ingress test:

```bash
curl http://shop.local/products
curl http://shop.local/orders
```

7. Explanation of:

- What Service does
- Difference between ClusterIP and NodePort
- Why Ingress is used
- Why databases should stay internal

---

# Final Summary

Kubernetes Pods are temporary and their IPs can change.

A Service gives a stable way to reach Pods.

ClusterIP is for internal communication.

NodePort is for simple external testing using node IP and port.

Ingress is for smart HTTP routing using domain names and paths.

In real projects:

```text
Internal backend/database -> ClusterIP
Temporary testing         -> NodePort
Web application routing   -> Ingress
```

Final memory trick:

```text
ClusterIP = Internal office room
NodePort  = Door in the building wall
Ingress   = Smart traffic guard
```
