
# Kubernetes Application Delivery (Real “Ops” Scenario)
**Deployments + Services + Ingress + Health Checks (readiness/liveness)**

This is my way of delivering an application in Kubernetes: I deploy it with a **Deployment**, expose it internally with a **Service**, publish it externally using **Ingress (NGINX)**, and I protect uptime with **readiness + liveness probes** so bad pods don’t take traffic.

---

## Problem

In production, “it works on my laptop” is not enough.

Common issues I’ve seen (and simulated here):

- The app starts, but it’s **not ready** yet and still receives traffic → users get errors.
- A container **hangs** or becomes unhealthy but stays running → slow outages.
- The app is deployed, but there’s **no clean external access** (no hostname/path routing).
- Debugging is hard without a clear delivery pattern and proof (CLI + screenshots).

---

## Solution

I built a delivery pattern that looks like a real platform setup:

- **Deployment** runs the app with **replicas** for availability
- **Service (ClusterIP)** provides stable internal networking
- **Ingress (NGINX)** exposes the app using a clean URL (host/path)
- **Readiness probe** prevents traffic until the app is ready
- **Liveness probe** restarts containers when the app becomes unhealthy
- **Rolling updates** so I can ship new versions with minimal downtime

---

## Architecture Diagram

![Architecture Diagram](screenshots/architecture.png)

---

## Step-by-step CLI

> Example namespace: `delivery-demo`  
> Example app name: `demo-app`  
> If you are using Minikube, enable ingress first.

### 1) Create a project folder + files

**Goal:** Keep everything clean and repeatable.

```bash
mkdir -p kubernetes-application-delivery/{k8s,screenshots}
cd kubernetes-application-delivery
touch README.md k8s/namespace.yaml k8s/deployment.yaml k8s/service.yaml k8s/ingress.yaml
````

---

### 2) Start Minikube and enable NGINX Ingress (Minikube only)

**Goal:** Ensure the Ingress Controller exists before applying Ingress rules.

```bash
minikube start
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

#### 📸 Screenshot 01: Ingress controller running

![Ingress controller running](screenshots/01-ingress-controller-running.png)

---

### 3) Create the namespace

**Goal:** Isolate the app like a real environment.

Create `k8s/namespace.yaml`:

```yaml

```

Apply:

```bash
kubectl apply -f k8s/namespace.yaml
kubectl get ns | grep delivery-demo
```

---

### 4) Deploy the application (Deployment + health checks)

**Goal:** Run multiple replicas and protect traffic using readiness/liveness probes.

Create `k8s/deployment.yaml`:

```yaml

```

Apply:

```bash
kubectl apply -f k8s/deployment.yaml
kubectl -n delivery-demo get deploy,pods -o wide
```

#### 📸 Screenshot 02: Deployment + pods ready

![Deployment + pods ready](screenshots/02-deployment-pods-ready.png)



---

### 5) Expose the app internally (Service)

**Goal:** Stable internal endpoint for the app. 

Create `k8s/service.yaml`:

```yaml

```

Apply:

```bash
kubectl apply -f k8s/service.yaml
kubectl -n delivery-demo get svc
```


#### 📸 Screenshot 03: Service created

![Service created](screenshots/03-service-created.png)


---

### 6) Publish externally (Ingress)

**Goal:** Clean external access using hostname/path routing.

Create `k8s/ingress.yaml`:

```yaml

```

Apply:

```bash
kubectl apply -f k8s/ingress.yaml
kubectl -n delivery-demo get ingress
kubectl -n delivery-demo describe ingress demo-app-ingress
```
#### 📸 Screenshot 04: Ingress created

![Ingress created](screenshots/04-ingress-created.png)



---

### 7) Map hostname to Minikube IP (local testing)

**Goal:** Prove Ingress routing works like a real domain.

Add to your local hosts file (Ubuntu WSL):

```bash
sudo vim /etc/hosts
127.0.0.1 demo-app.local

```

Add to your local hosts file (Windows example):

* Open Notepad as Admin
* Edit: `C:\Windows\System32\drivers\etc\hosts`
* Add:

  ```
  127.0.0.1 demo-app.local
  ```

Test:

```bash
curl -I http://demo-app.local/
```
#### 📸 Screenshot 05: Browser success

![Browser success](screenshots/05-browser-success.png)


---

### 8) Validate health checks are working

**Goal:** Confirm readiness/liveness are configured and pods are Ready.

Check probes:

```bash

kubectl -n delivery-demo describe pod demo-app-85cbf89dc9-gvxcq | sed -n '/Liveness:/,/Environment:/p'

kubectl -n delivery-demo get pods
```

#### 📸 Screenshot 06: Describe probes proof

![Describe probes](screenshots/06-describe-probes.png)



---

### 9) Simulate a “bad rollout” and watch Kubernetes protect traffic

**Goal:** Real ops behavior: safe rollouts + rollback in seconds.

Rollout a change (example):

```bash
kubectl -n delivery-demo set image deploy/demo-app demo-app=nginx:does-not-exist
kubectl -n delivery-demo rollout status deploy/demo-app
kubectl -n delivery-demo rollout history deploy/demo-app
```

#### 📸 Screenshot 07: Rollout status

![Rollout status](screenshots/07-rollout-status.png)


---

### 10) Events troubleshooting (when something goes wrong)

**Goal:** When it breaks, events tell you *exactly* what Kubernetes is unhappy about.

```bash
kubectl -n delivery-demo get events --sort-by=.metadata.creationTimestamp
```

#### 📸 Screenshot 08: Events troubleshooting

![Events troubleshooting](screenshots/08-events-troubleshooting.png)

Rollback if needed:

```bash
kubectl -n delivery-demo rollout undo deploy/demo-app
kubectl -n delivery-demo rollout status deploy/demo-app
```


---

## Outcome

* My app is deployed using a **repeatable delivery pattern**
* Users access it through a clean endpoint via **Ingress**
* **Readiness** prevents traffic to unready pods
* **Liveness** automatically restarts unhealthy containers
* I can do **safe rolling updates** and **rollback fast**
* I have proof (CLI + screenshots) like real ops documentation

---

## Troubleshooting

### 1) Ingress has no address / not routing

**Check ingress controller:**

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

**Check ingress details:**

```bash
kubectl -n delivery-demo describe ingress demo-app-ingress
kubectl -n delivery-demo get events --sort-by=.metadata.creationTimestamp
```

---

### 2) `curl demo-app.local` fails

**Confirm hosts mapping + Minikube IP:**

```bash
minikube ip
```

**Confirm ingress is created:**

```bash
kubectl -n delivery-demo get ingress
```

---

### 3) Pods are running but not Ready

**Readiness probe might be failing:**

```bash
kubectl -n delivery-demo describe pod -l app=demo-app
kubectl -n delivery-demo logs -l app=demo-app --tail=100
```

---

### 4) Service has no endpoints

Usually means selector doesn’t match pod labels.

```bash
kubectl -n delivery-demo describe svc demo-app-svc
kubectl -n delivery-demo get pods --show-labels
```

Make sure:

* Service selector: `app: demo-app`
* Pod labels: `app: demo-app`

---

### 5) Rollout stuck

```bash
kubectl -n delivery-demo rollout status deploy/demo-app
kubectl -n delivery-demo describe deploy demo-app
kubectl -n delivery-demo get events --sort-by=.metadata.creationTimestamp
```

Rollback:

```bash
kubectl -n delivery-demo rollout undo deploy/demo-app
```

---


