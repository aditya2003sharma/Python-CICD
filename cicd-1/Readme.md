# Python CI/CD Pipeline
### Jenkins + Docker + ArgoCD + Minikube (Kubernetes)

> A complete CI/CD pipeline for a Python Flask application — from code push to live deployment on Kubernetes.

---

## Pipeline Flow

```
Code Push → GitHub → Jenkins (CI) → Docker Hub → ArgoCD (CD) → Minikube
```

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Python 3.11 + Flask | Web application |
| pytest | Unit testing |
| Docker | Containerization |
| Jenkins | CI server (Build, Test, Push) |
| Docker Hub | Container image registry |
| ArgoCD | GitOps CD tool |
| Minikube | Local Kubernetes cluster |
| kubectl | Kubernetes CLI |

---

## Repository Structure

```
Python-CICD/
└── cicd-1/
    ├── app.py                  # Flask web application
    ├── requirements.txt        # Python dependencies
    ├── test_app.py             # Unit tests
    ├── Dockerfile              # Docker build instructions
    ├── Jenkinsfile             # Jenkins CI pipeline
    └── k8s-manifests/
        ├── deployment.yml      # Kubernetes Deployment
        └── service.yml         # Kubernetes Service
```

---

## Step 1 — Application Files

### app.py
```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return 'Hello from Python CI/CD Pipeline!'

@app.route('/health')
def health():
    return 'OK', 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### requirements.txt
```
flask==3.0.0
pytest==7.4.0
```

### test_app.py
```python
import pytest
from app import app

@pytest.fixture
def client():
    app.config['TESTING'] = True
    with app.test_client() as client:
        yield client

def test_home(client):
    res = client.get('/')
    assert res.status_code == 200

def test_health(client):
    res = client.get('/health')
    assert res.status_code == 200
```

### Dockerfile
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

---

## Step 2 — Kubernetes Manifests

### k8s-manifests/deployment.yml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: python-app
  template:
    metadata:
      labels:
        app: python-app
    spec:
      containers:
      - name: python-app
        image: aditya28jan2003/python-cicd:replaceImageTag
        ports:
        - containerPort: 5000
```

> `replaceImageTag` is automatically replaced by Jenkins with the `BUILD_NUMBER` on every build.

### k8s-manifests/service.yml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: python-app-service
  namespace: default
spec:
  selector:
    app: python-app
  ports:
  - port: 5000
    targetPort: 5000
  type: NodePort
```

---

## Step 3 — Jenkinsfile (CI Pipeline)

### Pipeline Stages

| Stage | What Happens |
|---|---|
| Checkout | Jenkins pulls latest code from GitHub |
| Install Dependencies | Runs `pip install -r requirements.txt` |
| Run Tests | Runs `pytest` — pipeline stops if tests fail |
| Build & Push Docker Image | Builds and pushes image to Docker Hub |
| Update Deployment File | Updates `deployment.yml` with new image tag and pushes to GitHub |

### Jenkinsfile
```groovy
pipeline {
  agent {
    docker {
      image 'python:3.11-slim'
      args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
    }
  }
  stages {
    stage('Checkout') {
      steps {
        echo 'Code checked out'
      }
    }
    stage('Install Dependencies') {
      steps {
        sh 'pip install -r cicd-1/requirements.txt'
      }
    }
    stage('Run Tests') {
      steps {
        sh 'pytest cicd-1/test_app.py -v'
      }
    }
    stage('Build and Push Docker Image') {
      environment {
        DOCKER_IMAGE = "aditya28jan2003/python-cicd:${BUILD_NUMBER}"
      }
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'docker-creds',
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh '''
            apt-get update && apt-get install -y docker.io
            docker login -u $DOCKER_USER -p $DOCKER_PASS
            docker build -t ${DOCKER_IMAGE} cicd-1/
            docker push ${DOCKER_IMAGE}
          '''
        }
      }
    }
    stage('Update Deployment File') {
      environment {
        GIT_REPO_NAME = "Python-CICD"
        GIT_USER_NAME = "aditya2003sharma"
      }
      steps {
        withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
          sh """
            apt-get install -y git
            git config --global --add safe.directory ${WORKSPACE}
            git config --global user.email "aditya28jan2003@gmail.com"
            git config --global user.name "Aditya Sharma"
            sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" cicd-1/k8s-manifests/deployment.yml
            git add cicd-1/k8s-manifests/deployment.yml
            git commit -m "Update image to version ${BUILD_NUMBER}" || true
            git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git HEAD:main
          """
        }
      }
    }
  }
}
```

### Jenkins Credentials Required

Go to **Jenkins → Manage Jenkins → Credentials → Global → Add Credential**

| Credential ID | Type | Value |
|---|---|---|
| `docker-creds` | Username & Password | Docker Hub login |
| `github` | Secret Text | GitHub Personal Access Token |

### Jenkins Job Setup
1. New Item → `python-cicd` → Pipeline → OK
2. Pipeline → Pipeline script from SCM → Git
3. Repository URL: `https://github.com/aditya2003sharma/Python-CICD`
4. Branch: `*/main`
5. Script Path: `cicd-1/Jenkinsfile`
6. Save → Build Now

---

## Step 4 — ArgoCD Setup

### What is ArgoCD?
ArgoCD is a GitOps tool for Kubernetes. It watches your GitHub repo and automatically syncs any changes to your Kubernetes cluster. Your Git repo is the single source of truth.

### How ArgoCD works in this project
1. Jenkins updates `deployment.yml` with new image tag and pushes to GitHub
2. ArgoCD detects the change in GitHub repo
3. ArgoCD compares GitHub state vs Kubernetes state
4. ArgoCD syncs the difference — pulls new Docker image and redeploys
5. New version is live with zero manual steps

### Install ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd -w
```
Wait until all pods show `Running` then press `Ctrl+C`

### Expose ArgoCD UI
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
minikube service argocd-server -n argocd --url
```

### Get Admin Password
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```
Login with **username:** `admin` and the password from above.

### Create ArgoCD Application
In the ArgoCD UI click **New App** and fill in:

| Field | Value |
|---|---|
| Application Name | `python-cicd` |
| Repository URL | `https://github.com/aditya2003sharma/Python-CICD` |
| Revision | `main` |
| Path | `cicd-1/k8s-manifests` |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `default` |

Click **Create** then **Sync**.

### ArgoCD Application States

| State | Meaning |
|---|---|
| `Synced` | GitHub matches what is running in Kubernetes |
| `OutOfSync` | GitHub has changes not yet deployed |
| `Healthy` | All pods running correctly |
| `Degraded` | Some pods are failing |
| `Progressing` | Deployment is rolling out |

### Enable Auto Sync (Recommended)
App Details → Sync Policy → Enable **Auto-Sync**

ArgoCD will automatically deploy every time Jenkins pushes a change to GitHub.

---

## Step 5 — Access the Application

### 1. Verify pods are running
```bash
kubectl get pods
```
Expected:
```
NAME                          READY   STATUS    RESTARTS   AGE
python-app-xxxxxxxxx-xxxxx    1/1     Running   0          1m
python-app-xxxxxxxxx-xxxxx    1/1     Running   0          1m
```

### 2. Verify service exists
```bash
kubectl get svc
```
Expected:
```
NAME                 TYPE       CLUSTER-IP   PORT(S)          AGE
python-app-service   NodePort   10.x.x.x     5000:3xxxx/TCP   1m
```

### 3. Get application URL
```bash
minikube service python-app-service --url
```

### 4. Test the application
```bash
curl http://<MINIKUBE_URL>
# Expected: Hello from Python CI/CD Pipeline!

curl http://<MINIKUBE_URL>/health
# Expected: OK
```

Or open the URL directly in your browser.

---

## End to End Flow

| # | Who | What Happens |
|---|---|---|
| 1 | Developer | Pushes code to GitHub main branch |
| 2 | Jenkins | Detects push and triggers pipeline |
| 3 | Jenkins | Installs dependencies inside Docker container |
| 4 | Jenkins | Runs pytest — stops if tests fail |
| 5 | Jenkins | Builds Docker image tagged with BUILD_NUMBER |
| 6 | Jenkins | Pushes image to Docker Hub |
| 7 | Jenkins | Updates deployment.yml with new image tag |
| 8 | Jenkins | Pushes updated deployment.yml to GitHub |
| 9 | ArgoCD | Detects change in GitHub repo |
| 10 | ArgoCD | Syncs new manifest to Minikube |
| 11 | Kubernetes | Pulls new Docker image from Docker Hub |
| 12 | Kubernetes | Rolls out new deployment |
| 13 | You | Visit app URL and see updated application |

---

## Quick Reference Commands

```bash
# Minikube
minikube start                             # Start cluster
minikube status                            # Check cluster status
minikube service python-app-service --url  # Get app URL
minikube service argocd-server -n argocd --url  # Get ArgoCD URL

# Kubernetes
kubectl get pods                           # Check pods
kubectl get svc                            # Check services
kubectl logs <pod-name>                    # View pod logs
kubectl describe pod <pod-name>            # Pod details

# ArgoCD
kubectl get pods -n argocd                 # ArgoCD pods status
```

---

**Author:** Aditya Sharma | aditya28jan2003@gmail.com | [github.com/aditya2003sharma](https://github.com/aditya2003sharma)
