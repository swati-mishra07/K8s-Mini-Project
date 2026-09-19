# K8s-Mini-Project

A small hands-on project for learning Kubernetes. A simple **Flask** web app is containerized with **Docker** and deployed to a local Kubernetes cluster using a `Deployment` (2 replicas) and a `Service`.

![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?logo=kubernetes&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-000000?logo=flask&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green)

---

## Table of Contents

- [What This Project Covers](#what-this-project-covers)
- [Architecture](#architecture)
- [Repository Structure](#repository-structure)
- [The Application](#the-application)
- [Docker Image](#docker-image)
- [Kubernetes Manifest](#kubernetes-manifest)
- [Prerequisites](#prerequisites)
- [Run Locally (without Docker)](#run-locally-without-docker)
- [Deploy to Kubernetes](#deploy-to-kubernetes)
- [Useful kubectl Commands](#useful-kubectl-commands)
- [Cleanup](#cleanup)
- [Troubleshooting](#troubleshooting)
- [Future Improvements](#future-improvements)
- [License](#license)

---

## What This Project Covers

- Writing a small Flask app with a form that accepts user input
- Packaging it into a container image with a Dockerfile
- Running multiple replicas of the app with a Kubernetes `Deployment`
- Setting CPU and memory limits on containers
- Exposing the pods inside the cluster with a `Service`
- Reaching the app from your machine with `kubectl port-forward`

---

## Architecture

```
                 kubectl port-forward
   Browser  ───────────────────────────►  Service: kubernetes-test-app
 localhost:8080                                  port 8080
                                                    │
                                                    ▼ targetPort 5000
                                     ┌──────────────┴──────────────┐
                                     ▼                             ▼
                              Pod (replica 1)               Pod (replica 2)
                            Flask app :5000               Flask app :5000
```

The Service load-balances traffic across both pods using the label selector `app: kubernetes-test-app`.

---

## Repository Structure

```
K8s-Mini-Project/
├── app.py              # Flask application
├── templates/          # HTML templates (index.html is rendered by app.py)
├── static/             # Static files served by Flask
├── requirements.txt    # Python dependencies (Flask==2.3.2)
├── dockerfile          # Container image definition
├── deployment.yaml     # Kubernetes Deployment + Service
├── LICENSE             # MIT license
└── README.md
```

---

## The Application

`app.py` is a single-route Flask app listening on port `5000` (`0.0.0.0`).

| Route | Method | Behavior |
|-------|--------|----------|
| `/` | `GET` | Renders `index.html` with an empty message |
| `/` | `POST` | Reads the `name` form field and renders `index.html` with the message `Hello <name>, Welcome to the Kubernetes test application!!!` |

Submit your name in the form on the home page and the greeting appears on the page.

---

## Docker Image

The `dockerfile`:

- Uses the lightweight `python:3.12-slim-bookworm` base image
- Sets `PYTHONDONTWRITEBYTECODE=1` (no `.pyc` files) and `PYTHONUNBUFFERED=1` (logs show up immediately in `kubectl logs`)
- Copies only what the app needs: `requirements.txt`, `app.py`, `templates/` and `static/`
- Installs dependencies with `pip install --no-cache-dir` to keep the image small
- Exposes port `5000` and starts the app with `python app.py`

---

## Kubernetes Manifest

`deployment.yaml` defines two resources.

**Deployment `kubernetes-test-app`**

| Setting | Value |
|---------|-------|
| Replicas | `2` |
| Image | `kubernetes-test-app:latest` |
| `imagePullPolicy` | `Never` (uses the locally loaded image, no registry needed) |
| Container port | `5000` |
| Resource limits | `64Mi` memory, `200m` CPU |

**Service `kubernetes-test-app`**

| Setting | Value |
|---------|-------|
| Type | `ClusterIP` (default, since no type is specified) |
| Port | `8080` |
| Target port | `5000` |
| Selector | `app: kubernetes-test-app` |

> Because the Service is `ClusterIP`, it is only reachable inside the cluster. Use `kubectl port-forward` (below) to open it from your machine.

---

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- A local Kubernetes cluster, such as [Minikube](https://minikube.sigs.k8s.io/docs/start/) (commands below use Minikube)

---

## Run Locally (without Docker)

```bash
pip install -r requirements.txt
python app.py
```

Open `http://localhost:5000`.

---

## Deploy to Kubernetes

1. **Start the cluster**

   ```bash
   minikube start
   ```

2. **Build the Docker image**

   ```bash
   docker build -t kubernetes-test-app:latest -f dockerfile .
   ```

3. **Load the image into Minikube**

   ```bash
   minikube image load kubernetes-test-app:latest
   ```

   This is required because the Deployment uses `imagePullPolicy: Never`, so the image must already exist inside the cluster.

4. **Apply the manifest**

   ```bash
   kubectl apply -f deployment.yaml
   ```

5. **Check that everything is running**

   ```bash
   kubectl get deployments
   kubectl get pods
   kubectl get svc
   ```

   You should see 2 pods in `Running` state.

6. **Access the app**

   ```bash
   kubectl port-forward svc/kubernetes-test-app 8080:8080
   ```

   Open `http://localhost:8080`, enter your name and submit the form.

---

## Useful kubectl Commands

```bash
# Logs from one of the pods
kubectl logs <pod-name>

# Describe a pod (events, limits, image, etc.)
kubectl describe pod <pod-name>

# Scale the deployment
kubectl scale deployment kubernetes-test-app --replicas=3

# Watch pods get recreated after deleting one
kubectl delete pod <pod-name>
kubectl get pods -w
```

---

## Cleanup

```bash
kubectl delete -f deployment.yaml
minikube stop
```

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Pod status `ErrImageNeverPull` | The image is not inside the cluster. Re-run `minikube image load kubernetes-test-app:latest`. |
| Pod is `OOMKilled` or restarting | The `64Mi` memory limit is tight. Raise it in `deployment.yaml` and re-apply. |
| `localhost:8080` doesn't load | Make sure the `kubectl port-forward` command is still running in its terminal. |
| `minikube service` doesn't open the app | The Service is `ClusterIP`. Use port-forward, or change the Service `type` to `NodePort`. |

---

## Future Improvements

- Add liveness and readiness probes
- Expose the app with a `NodePort` service or an Ingress
- Move configuration into a `ConfigMap`
- Add a Horizontal Pod Autoscaler
- Push the image to a registry and drop `imagePullPolicy: Never`
- Add a screenshot of the running app

---

## License

This project is licensed under the [MIT License](LICENSE).
