# containerized-backend-app-eks
Containerized a Flask backend, pushed the image to Amazon ECR, and deployed it to an Amazon EKS cluster using Kubernetes Deployment and Service manifests, managed with kubectl.
# Deploy a Flask Backend on Amazon EKS with Kubernetes

Containerized a Flask backend, pushed the image to Amazon ECR, and deployed it to an Amazon EKS cluster using Kubernetes Deployment and Service manifests, managed with `kubectl`.

![AWS](https://img.shields.io/badge/AWS-EKS%20%7C%20ECR%20%7C%20IAM-orange)
![Kubernetes](https://img.shields.io/badge/Kubernetes-kubectl-326CE5)
![Docker](https://img.shields.io/badge/Docker-containerized-2496ED)

> Guided project built with [NextWork](https://nextwork.ai). Project link: [View Project](https://nextwork.ai/projects/090a4709-f1f4-5d95-ae4d-b8199bf808ed)

---

## Overview

| Item | Detail |
|---|---|
| **Goal** | Deploy a containerized backend app on a managed Kubernetes cluster |
| **App** | `nextwork-flask-backend` (Python Flask) |
| **Cluster** | `nextwork-eks-cluster` (Amazon EKS, `us-east-1`) |
| **Registry** | Amazon ECR |
| **Tooling** | Docker, AWS CLI, eksctl, kubectl |
| **Hardest part** | Fixing YAML indentation errors in the manifest files |

## Architecture / Workflow

```
Git repo (Flask backend)
        |
        v
  docker build  -->  Container image
        |
        v
  docker push   -->  Amazon ECR repository
        |
        v
  kubectl apply -f deployment.yaml / service.yaml
        |
        v
  Amazon EKS cluster
   |-- Deployment  -> manages ReplicaSet -> Pods (containers pulled from ECR)
   '-- Service     -> exposes the backend outside the cluster
```

## Key Concepts

- **EKS cluster:** The managed Kubernetes control plane plus worker nodes (EC2 instances). It schedules, starts, stops, and scales containers across nodes.
- **Container image:** A packaged build of the app. Kubernetes runs containers from images, so the image must exist in a registry the cluster can reach.
- **Amazon ECR:** AWS container registry. Nodes pull the image from here, keeping every node on the same version.
- **Manifest (YAML):** A declarative file that tells Kubernetes what to run, how many copies, and how to expose it.
- **Deployment:** A Kubernetes resource that creates and replaces pods, scales replicas, and handles rolling updates.
- **Service:** A Kubernetes resource that gives the pods a stable network endpoint so the backend is reachable from outside the cluster.
- **Pod:** The smallest deployable unit in Kubernetes. Containers always run inside a pod.
- **kubectl:** The command-line tool for applying manifests and inspecting cluster resources.

## Prerequisites

- AWS account with permissions for EKS, ECR, and IAM
- AWS CLI configured (`aws configure`)
- Docker, `kubectl`, and `eksctl` installed
- An existing EKS cluster (`nextwork-eks-cluster`) with worker nodes
- Backend source code cloned from Git

## Steps

### 1. Clone the backend code

```bash
git clone <BACKEND_REPO_URL>
cd <BACKEND_REPO_DIRECTORY>
```

### 2. Build the container image

```bash
docker build -t nextwork-flask-backend .
```

### 3. Push the image to ECR

```bash
# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com

# Create the repository (one time)
aws ecr create-repository --repository-name nextwork-flask-backend --region us-east-1

# Tag and push
docker tag nextwork-flask-backend:latest <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/nextwork-flask-backend:latest
docker push <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/nextwork-flask-backend:latest
```

### 4. Write the Kubernetes manifests

The files below are templates. Adjust names, ports, and replica counts to match your actual manifests.

`manifests/flask-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextwork-flask-backend
spec:
  replicas: 2                      # adjust as needed
  selector:
    matchLabels:
      app: nextwork-flask-backend
  template:
    metadata:
      labels:
        app: nextwork-flask-backend
    spec:
      containers:
        - name: nextwork-flask-backend
          image: <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/nextwork-flask-backend:latest
          ports:
            - containerPort: <APP_PORT>   # port your Flask app listens on
```

`manifests/flask-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nextwork-flask-backend
spec:
  type: LoadBalancer
  selector:
    app: nextwork-flask-backend
  ports:
    - port: 80
      targetPort: <APP_PORT>
```

### 5. Deploy with kubectl

```bash
kubectl apply -f manifests/flask-deployment.yaml
kubectl apply -f manifests/flask-service.yaml
```

### 6. Verify the deployment

```bash
kubectl get deployments
kubectl get pods
kubectl get svc nextwork-flask-backend
kubectl describe pod <POD_NAME>
kubectl logs <POD_NAME>
```

## Extension: Verify in the EKS Console

To view workloads in the AWS console, the IAM identity needs access to the cluster. I mapped my IAM user to the cluster with:

```bash
eksctl create iamidentitymapping \
  --cluster nextwork-eks-cluster \
  --arn arn:aws:iam::<ACCOUNT_ID>:user/<IAM_USER> \
  --group system:masters \
  --username admin \
  --region us-east-1
```

> **Note:** `system:masters` grants full cluster-admin rights. That is fine for a learning lab, but in real environments use a least-privilege role with scoped RBAC permissions or EKS access entries.

After access was granted, the EKS console showed the pods running on each node and the events recorded for each pod.

## Screenshots

Save your screenshots in an `images/` folder and reference them like this:

```markdown
![Manifest deployment](images/manifest-deployment.png)
![kubectl apply output](images/kubectl-apply.png)
![EKS console pods and events](images/eks-pods-events.png)
```

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `error converting YAML to JSON` or `mapping values are not allowed` | Wrong indentation or tabs in YAML | Use 2 spaces, never tabs. Validate with `kubectl apply --dry-run=client -f <file>` |
| Pod in `ImagePullBackOff` | Wrong image URI or no ECR pull permission | Check the image URI and tag; confirm the node role has ECR read access |
| Pod in `CrashLoopBackOff` | App error or wrong port | Run `kubectl logs <pod>` and check `containerPort` |
| Console shows "no access to resources" | IAM identity not mapped to the cluster | Run the `eksctl create iamidentitymapping` command above |
| Service has no external address | Load balancer still provisioning | Wait a few minutes, then re-run `kubectl get svc` |

## Cleanup (avoid ongoing AWS charges)

EKS clusters, worker nodes, and load balancers all incur cost. Delete them when you are done:

```bash
kubectl delete -f manifests/
eksctl delete cluster --name nextwork-eks-cluster --region us-east-1
aws ecr delete-repository --repository-name nextwork-flask-backend --force --region us-east-1
```

## What I Learned

- How the build, push, and deploy flow works from source code to running pods on EKS
- The difference between a Deployment (manages pods and updates) and a Service (exposes them)
- How to inspect pods, logs, and events with `kubectl` and the EKS console
- How IAM identities are mapped to Kubernetes access with `eksctl`
- How to debug YAML indentation errors



## Author

**Imayavarman Senthilkumar**
Email: i.varman.work@gmail.com

---

*Built with [NextWork](https://nextwork.ai)*
