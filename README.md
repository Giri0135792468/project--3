# Kubernetes Hands-On Project 3 — Kubernetes Probes

## 1. Project Objective

Build and deploy a web application on Amazon EKS and understand Kubernetes container health checks through hands-on failure testing.

This project introduces:

* Readiness Probe
* Liveness Probe
* Startup Probe
* HTTP probes
* Probe failure behavior
* `Running` vs `Ready`
* Container restart behavior
* `CrashLoopBackOff`
* Rolling update behavior when a new Pod is not Ready
* How Startup Probe interacts with Liveness and Readiness

The application is developed and tested locally with Docker before being deployed to EKS.

---

# 2. Project Workflow

```text
VS Code
   ↓
Develop Application
   ↓
Docker Build
   ↓
Docker Test Locally
   ↓
Docker Hub
   ↓
GitHub
   ↓
EKS
   ↓
Deployment
   ↓
Readiness Probe
   ↓
Liveness Probe
   ↓
Startup Probe
   ↓
Failure Testing
```

---

# 3. Project Structure

```text
project-3/
│
├── README.md
├── index.html
├── Dockerfile
│
└── k8s/
    └── deployment.yaml
```

---

# 4. Application

The application is a simple Nginx web application.

## index.html

```html
<!DOCTYPE html>
<html>
<head>
    <title>Kubernetes Project 3</title>
</head>
<body>
    <h1>Project 3 - Kubernetes Probes</h1>

    <p>Application: Probe Demo</p>
    <p>Version: 1.0</p>
    <p>Status: Healthy</p>
</body>
</html>
```

---

# 5. Dockerfile

```dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
```

---

# 6. Docker Image

Build:

```bash
docker build -t k8s-project-3:v1 .
```

Tag:

```bash
docker tag k8s-project-3:v1 <dockerhub-username>/k8s-project-3:v1
```

Push:

```bash
docker push <dockerhub-username>/k8s-project-3:v1
```

---

# 7. Local Docker Testing

Run:

```bash
docker run -d --name k8s-project-3 -p 8083:80 k8s-project-3:v1
```

Test:

```bash
curl http://localhost:8083
```

The application was successfully tested locally before being deployed to EKS.

---

# 8. Kubernetes Namespace

The application runs in:

```text
project-3
```

Create:

```bash
kubectl create namespace project-3
```

Verify:

```bash
kubectl get namespace project-3
```

---

# 9. Deployment

The Deployment manages two application Pods.

## deployment.yaml

The final Deployment contains all three probes:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: project-3-app
  namespace: project-3

spec:
  replicas: 2

  selector:
    matchLabels:
      app: project-3

  template:
    metadata:
      labels:
        app: project-3

    spec:
      containers:
        - name: nginx
          image: <dockerhub-username>/k8s-project-3:v1

          ports:
            - containerPort: 80

          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10

          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10

          startupProbe:
            httpGet:
              path: /
              port: 80
            failureThreshold: 30
            periodSeconds: 2
```

Apply:

```bash
kubectl apply -f k8s/deployment.yaml
```

Verify:

```bash
kubectl get deployment -n project-3
kubectl get pods -n project-3
```

---

# 10. HTTP Probe

All three probes in this project use an HTTP GET request.

For example:

```yaml
httpGet:
  path: /
  port: 80
```

Kubernetes sends an HTTP request to:

```text
http://<Pod-IP>:80/
```

If the HTTP response is successful, the probe succeeds.

If the request fails, Kubernetes considers the probe unsuccessful.

---

# 11. Readiness Probe

## Purpose

A Readiness Probe answers:

> **Is this Pod ready to receive traffic?**

Configuration:

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
```

### Behavior

```text
Readiness Probe succeeds
        ↓
Pod becomes Ready
        ↓
Pod can receive Service traffic
```

If the readiness probe fails:

```text
Pod remains Running
        ↓
Pod becomes Not Ready
        ↓
Pod is removed from Service endpoints
        ↓
No new traffic is sent to that Pod
```

### Important

A readiness failure **does not restart the container**.

---

# 12. Readiness Probe Hands-On Test

Initially the application had:

```text
READY   STATUS
1/1     Running
1/1     Running
```

The readiness probe was then intentionally changed from:

```yaml
path: /
```

to:

```yaml
path: /wrong
```

Because Nginx does not have that path, the HTTP request returned an unsuccessful response.

The new Pod became:

```text
0/1   Running
```

This demonstrated:

```text
Container = Running
Application = Not Ready
```

The old healthy Pods continued running while the new Pod was not Ready.

---

# 13. Readiness Probe and Rolling Updates

During the rolling update we observed:

```text
Old ReplicaSet
├── Pod 1 → 1/1 Running
└── Pod 2 → 1/1 Running

New ReplicaSet
└── Pod 3 → 0/1 Running
```

The new Pod was running but not Ready.

Kubernetes therefore did not immediately remove all healthy old Pods.

Conceptually:

```text
New Pod
   ↓
Not Ready
   ↓
Old healthy Pods remain available
```

This helps maintain application availability during rolling updates.

The exact rollout behavior is controlled by the Deployment's rolling-update settings such as `maxUnavailable` and `maxSurge`.

---

# 14. Running vs Ready

This was one of the most important concepts demonstrated in Project 3.

```text
Running
```

means the container is running.

```text
Ready
```

means Kubernetes considers the Pod ready to receive traffic.

Therefore:

```text
Running ≠ Ready
```

Example:

```text
0/1 Running
```

means:

```text
Container is running
but
Pod is not Ready
```

---

# 15. Liveness Probe

## Purpose

A Liveness Probe answers:

> **Is the container still healthy enough to continue running, or should Kubernetes restart it?**

Configuration:

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
```

### Behavior

```text
Liveness Probe succeeds
        ↓
Container continues running
```

If the liveness probe repeatedly fails:

```text
Liveness Probe fails
        ↓
Container considered unhealthy
        ↓
Kubernetes restarts the container
        ↓
Restart count increases
```

---

# 16. Liveness Probe Hands-On Test

The liveness probe was intentionally changed from:

```yaml
path: /
```

to:

```yaml
path: /wrong
```

Nginx returned an unsuccessful HTTP response.

Kubernetes repeatedly failed the liveness probe.

We observed:

```text
RESTARTS
5
```

and eventually:

```text
CrashLoopBackOff
```

---

# 17. CrashLoopBackOff

The test demonstrated:

```text
Liveness Probe fails
        ↓
Container restarted
        ↓
Liveness Probe fails again
        ↓
Container restarted again
        ↓
Repeated failures
        ↓
CrashLoopBackOff
```

Important:

> `CrashLoopBackOff` does not necessarily mean the application itself crashed.

In our test, the container was being repeatedly restarted because the liveness probe continuously failed.

---

# 18. Readiness vs Liveness

This is one of the most important interview concepts.

| Probe     | Failure behavior                               |
| --------- | ---------------------------------------------- |
| Readiness | Pod becomes Not Ready                          |
| Readiness | Pod stops receiving Service traffic            |
| Readiness | Container is not restarted                     |
| Liveness  | Container is restarted                         |
| Liveness  | Restart count can increase                     |
| Liveness  | Repeated failures can lead to CrashLoopBackOff |

Simple memory:

```text
Readiness → Should I receive traffic?

Liveness → Should I be restarted?
```

---

# 19. Startup Probe

## Purpose

A Startup Probe answers:

> **Has the application successfully finished starting?**

It is particularly useful for applications that take a long time to initialize.

Configuration:

```yaml
startupProbe:
  httpGet:
    path: /
    port: 80
  failureThreshold: 30
  periodSeconds: 2
```

---

# 20. Startup Probe Timing

The configuration:

```yaml
failureThreshold: 30
periodSeconds: 2
```

allows approximately:

```text
30 × 2 seconds
=
60 seconds
```

for the application to successfully pass the startup probe.

---

# 21. Startup Probe Relationship

The conceptual flow is:

```text
Container starts
       ↓
Startup Probe
       ↓
Startup succeeds
       ↓
Liveness + Readiness can begin their normal checks
```

While the Startup Probe has not yet succeeded, Kubernetes does not use the Liveness and Readiness probes to make their normal health decisions for that container.

This prevents a slow-starting application from being restarted prematurely by its Liveness Probe.

---

# 22. Startup Probe Hands-On Test

The Startup Probe was intentionally changed from:

```yaml
path: /
```

to:

```yaml
path: /wrong
```

The new Pod became:

```text
0/1   Running
```

while the existing healthy Pods remained:

```text
1/1   Running
1/1   Running
```

The Startup Probe continued failing because:

```text
/wrong
```

did not return a successful HTTP response.

We also observed that the Liveness Probe did not immediately take over while the Startup Probe had not successfully completed.

---

# 23. Probe Failure Testing Summary

We intentionally broke each probe.

### Readiness failure

```text
Readiness Probe
      ↓
Fails
      ↓
Pod = 0/1 Running
      ↓
Pod stays running
      ↓
Pod is not considered Ready
```

### Liveness failure

```text
Liveness Probe
      ↓
Fails repeatedly
      ↓
Container restarted
      ↓
RESTARTS increases
      ↓
Possible CrashLoopBackOff
```

### Startup failure

```text
Startup Probe
      ↓
Fails
      ↓
Application has not passed startup
      ↓
Liveness/Readiness normal checks are held back
      ↓
If startup continues failing
      ↓
Container eventually restarted
```

---

# 24. Final Probe Architecture

```text
                    Container
                       │
                       ▼
                Startup Probe
                       │
                ┌──────┴──────┐
                │             │
             Success        Failure
                │             │
                ▼             ▼
       Liveness + Readiness   Keep
          normal checks       starting
                │
        ┌───────┴────────┐
        │                │
        ▼                ▼
   Liveness          Readiness
        │                │
        ▼                ▼
 Container restart    Traffic decision
 if unhealthy         if not ready
```

---

# 25. Final Application Flow

```text
                         GitHub
                           │
                           ▼
                      EKS Server
                           │
                           ▼
                  project-3 Namespace
                           │
                           ▼
                      Deployment
                           │
                     ReplicaSet
                           │
                ┌──────────┴──────────┐
                ▼                     ▼
              Pod 1                 Pod 2
                │                     │
                ├── Startup Probe     ├── Startup Probe
                │                     │
                ├── Readiness Probe   ├── Readiness Probe
                │                     │
                └── Liveness Probe    └── Liveness Probe
```

---

# 26. Concepts Covered

| Concept                                   | Hands-on |
| ----------------------------------------- | -------- |
| Namespace                                 | ✅        |
| Deployment                                | ✅        |
| ReplicaSet                                | ✅        |
| Pods                                      | ✅        |
| HTTP Probe                                | ✅        |
| Readiness Probe                           | ✅        |
| Readiness Failure                         | ✅        |
| Running vs Ready                          | ✅        |
| Rolling Update + Readiness                | ✅        |
| Liveness Probe                            | ✅        |
| Liveness Failure                          | ✅        |
| Container Restart                         | ✅        |
| Restart Count                             | ✅        |
| CrashLoopBackOff                          | ✅        |
| Startup Probe                             | ✅        |
| Startup Failure                           | ✅        |
| Startup → Liveness/Readiness relationship | ✅        |
| Docker                                    | ✅        |
| Docker Hub                                | ✅        |
| GitHub                                    | ✅        |
| EKS                                       | ✅        |

---

# 27. Concepts Intentionally Saved for Later

These are not part of Project 3:

```text
Resource Requests
Resource Limits
QoS Classes
HPA
Scheduling
Node Affinity
Taints
Tolerations
PersistentVolume
PersistentVolumeClaim
StorageClass
EBS CSI
EFS CSI
RBAC
ServiceAccount
IAM / Pod Identity
NetworkPolicy
Ingress
AWS Load Balancer Controller
ALB
StatefulSet
DaemonSet
Jobs
CronJobs
PDB
```

These will be introduced in later projects.

---

# 28. Project 3 Completion

Project 3 demonstrates how Kubernetes determines:

```text
Has the application started?
        ↓
Startup Probe

Is the application ready for traffic?
        ↓
Readiness Probe

Is the running application still healthy?
        ↓
Liveness Probe
```

The project is complete.
