# Practicals 1–13 — Copy-Paste Execution Guide

Oct 31, 2026 · @Tejas

## How to use this guide + one-time setup

Every command below runs in your **WSL Ubuntu terminal** with **Docker Desktop**, and all practicals share one Flask app in `~/devops-lab`. Each grey block is copied whole and pasted once — blocks that start with `cat > file <<'EOF'` create that file, so copy them all the way down to the closing `EOF`.

Ports used: the app runs on **8000** (your existing `studentdock-web` container already holds 5000), Jenkins 8080, Grafana 3000, Prometheus 9090, Kubernetes 30080.

### Step 1 — Connect Docker Desktop to WSL (once)

1. Start Docker Desktop and wait until it says *Engine running*.
2. Docker Desktop → **Settings → Resources → WSL integration** → turn on **Ubuntu** → **Apply & restart**.
3. Open the **Ubuntu** app from the Start menu (or type `wsl -d Ubuntu` in PowerShell).

### Step 2 — Install the base packages in Ubuntu (once)

```bash
sudo apt update
sudo apt install -y python3-venv python3-pip git curl unzip
```

### Step 3 — Check everything is ready

```bash
docker version --format 'Docker {{.Server.Version}}'
docker compose version
python3 --version
git --version
```

Expected: a Docker version (not an error), `Docker Compose version v2.x` or newer, `Python 3.12.x`, `git version 2.x`.

### Step 4 — Create the Flask app used by all practicals

A small Flask app (about 20 lines) with one page that shows a message, the version and the container/pod name, plus a `/health` route and 2 tests. Practicals 2–10 all reuse it.

```bash
mkdir -p ~/devops-lab/templates && cd ~/devops-lab

cat > app.py <<'EOF'
import os, socket
from flask import Flask, render_template

app = Flask(__name__)
VERSION = 'v1'


@app.route('/')
def home():
    return render_template('index.html', version=VERSION,
                           message=os.getenv('MESSAGE', 'Hello from Flask!'),
                           host=socket.gethostname())


@app.route('/health')
def health():
    return 'Server is up and running'


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

cat > templates/index.html <<'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>DevOps Lab</title>
    <style>
        body { background: linear-gradient(135deg, #0f0c29, #302b63); color: #fff;
               font-family: Arial; text-align: center; padding-top: 100px; }
        .card { display: inline-block; background: rgba(255,255,255,0.1);
                border-radius: 12px; padding: 20px 30px; margin: 10px; }
    </style>
</head>
<body>
    <h1>DevOps Lab App</h1>
    <h3>{{ message }}</h3>
    <div class="card">Version<br><b>{{ version }}</b></div>
    <div class="card">Container / Pod<br><b>{{ host }}</b></div>
</body>
</html>
EOF

cat > requirements.txt <<'EOF'
flask
pytest
EOF

cat > test_app.py <<'EOF'
from app import app


def test_home():
    assert app.test_client().get('/').status_code == 200


def test_health():
    assert b'up and running' in app.test_client().get('/health').data
EOF
```

Install the dependencies and run the tests:

```bash
cd ~/devops-lab
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
pytest -v
```

Expected: `2 passed`.

Run the app and open **http://localhost:8000** in your Windows browser (press `Ctrl+C` in the terminal to stop it):

```bash
flask --app app run --host 0.0.0.0 --port 8000
```

## Practical 2 & 3 — Git + GitHub: configure, branch, merge, pull request

Practical 2 (install/configure) and 3 (repositories) are done together, because Git is already installed in your Ubuntu.

### Step 1 — Configure Git

```bash
git --version
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
git config --global init.defaultBranch main
git config --list
```

### Step 2 — Connect to GitHub with an SSH key

Press **Enter** 3 times when `ssh-keygen` asks questions, then copy the line that `cat` prints.

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
cat ~/.ssh/id_ed25519.pub
```

GitHub → **Settings → SSH and GPG keys → New SSH key** → paste → **Add SSH key**. Then test (type `yes` if asked):

```bash
ssh -T git@github.com
```

Expected: `Hi <username>! You've successfully authenticated, but GitHub does not provide shell access.`

### Step 3 — Create the repository and push the app

On GitHub click **New repository** → name it `devops-lab` → **Public** → do NOT add a README → **Create repository**. Then change the first line to your GitHub username and paste:

```bash
GH_USER=your-github-username

cd ~/devops-lab
cat > .gitignore <<'EOF'
venv/
__pycache__/
.pytest_cache/
EOF

git init
git add .
git commit -m "First commit: Flask app"
git branch -M main
git remote add origin git@github.com:$GH_USER/devops-lab.git
git push -u origin main
```

Refresh GitHub — you will see `app.py`, `templates/`, `requirements.txt`, `test_app.py`.

### Step 4 — Create a branch, change code, push it

```bash
git checkout -b feature-update
sed -i 's/Hello from Flask!/Hello from feature branch!/' app.py
git add app.py
git commit -m "Update home page message"
git push -u origin feature-update
```

### Step 5 — Pull request and merge (on GitHub)

1. Open your repo → click **Compare & pull request** → **Create pull request**.
2. Open the **Files changed** tab to show the diff.
3. Click **Merge pull request → Confirm merge**.

### Step 6 — Bring the merge back to your machine

```bash
git checkout main
git pull origin main
git branch -d feature-update
git log --oneline --graph
```

Expected: the graph shows `Merge pull request #1 ...` joining your feature commit into main.

### Step 7 — Create and resolve a merge conflict

```bash
git checkout -b feature-title
sed -i 's/DevOps Lab App/DevOps Lab App (Feature)/' templates/index.html
git commit -am "Change title in feature"

git checkout main
sed -i 's/DevOps Lab App/DevOps Lab App (Main)/' templates/index.html
git commit -am "Change title in main"

git merge feature-title
```

Expected: `CONFLICT (content): Merge conflict in templates/index.html`. Show the conflict markers:

```bash
git diff
```

Resolve it by keeping the feature version, then push:

```bash
git checkout --theirs templates/index.html
git add templates/index.html
git commit -m "Resolve merge conflict"
git log --oneline --graph
git push origin main
```

Expected: the log shows the two branches joining at `Resolve merge conflict`.

## Practical 4 — Automating build and test using Jenkins

Jenkins runs in Docker, pulls your GitHub repo, creates a venv (Build) and runs pytest (Test).

### Step 1 — Add a Jenkinsfile to the repo

```bash
cd ~/devops-lab
cat > Jenkinsfile <<'EOF'
pipeline {
    agent any
    triggers { pollSCM('* * * * *') }
    stages {
        stage('Build') {
            steps {
                sh 'python3 -m venv venv'
                sh '. venv/bin/activate && pip install -r requirements.txt'
            }
        }
        stage('Test') {
            steps {
                sh '. venv/bin/activate && pytest -v'
            }
        }
    }
    post {
        success { echo 'Build and tests passed!' }
        failure { echo 'Build failed!' }
    }
}
EOF

git add Jenkinsfile
git commit -m "Add Jenkinsfile"
git push origin main
```

### Step 2 — Start Jenkins and give it Python

```bash
docker run -d -p 8080:8080 -p 50000:50000 --name jenkins -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts
docker exec -u root jenkins bash -c "apt-get update && apt-get install -y python3 python3-venv"
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

The last command prints the unlock password. Wait about 30 seconds after starting before running it.

### Step 3 — Set up Jenkins (browser)

1. Open **http://localhost:8080** → paste the password → **Install suggested plugins** (takes 2–3 min).
2. Create the admin user → **Save and Finish** → **Start using Jenkins**.
3. **New Item** → name `flask-pipeline` → choose **Pipeline** → **OK**.
4. Scroll to **Pipeline** → Definition: **Pipeline script from SCM** → SCM: **Git**.
5. Repository URL: `https://github.com/<your-github-username>/devops-lab.git` → Branch: `*/main` → Script Path: `Jenkinsfile` → **Save**.
6. Click **Build Now** → click build **#1** → **Console Output**.

Expected (end of the console output):

```text
test_app.py::test_home PASSED
test_app.py::test_health PASSED
============ 2 passed ============
Build and tests passed!
Finished: SUCCESS
```

### Step 4 — Show the automatic trigger

After build #1, Jenkins checks GitHub every minute. Push a change and a new build starts by itself within about a minute:

```bash
cd ~/devops-lab
echo "# DevOps Lab" > README.md
git add README.md
git commit -m "Trigger Jenkins build"
git push origin main
```

When you're done: `docker stop jenkins` (start again later with `docker start jenkins`).

## Practical 5 — CI/CD pipeline using GitHub Actions

On every push to `main`, GitHub installs the app and runs the tests; if they pass, a second job builds the Docker image, starts it and checks `/health`.

### Step 1 — Create the workflow file

```bash
cd ~/devops-lab
mkdir -p .github/workflows
cat > .github/workflows/ci-cd.yml <<'EOF'
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5

      - uses: actions/setup-python@v6
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest -v

  docker-build:
    needs: build-and-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5

      - name: Build Docker image
        run: docker build -t flask-app:latest .

      - name: Run container and test it
        run: |
          docker run -d -p 8000:5000 flask-app:latest
          sleep 5
          curl http://localhost:8000/health
EOF
```

The `docker-build` job needs the Dockerfile, so create it now (it is explained in Practical 7):

```bash
cat > Dockerfile <<'EOF'
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
EXPOSE 5000
CMD ["python", "app.py"]
EOF

cat > .dockerignore <<'EOF'
venv
.git
EOF
```

### Step 2 — Push and watch it run

```bash
git add .github Dockerfile .dockerignore
git commit -m "Add GitHub Actions CI/CD pipeline"
git push origin main
```

Open your repo → **Actions** tab → click the running workflow.

Expected: two green jobs, `build-and-test` → `docker-build`. Inside **Run tests** you see `2 passed`; inside **Run container and test it** you see `Server is up and running`.

To show a failing pipeline (optional), break a test, push, see the red cross, then undo it:

```bash
sed -i 's/up and running/DOWN/' test_app.py
git commit -am "Break test" && git push origin main

sed -i 's/DOWN/up and running/' test_app.py
git commit -am "Fix test" && git push origin main
```

## Practical 6 & 7 — Docker basic commands + Dockerfile and image

Docker is already installed through Docker Desktop, so Practical 6 (basic commands) and 7 (Dockerfile) are done together.

### Step 1 — Verify Docker

```bash
docker --version
docker run hello-world
```

Expected: `Hello from Docker! This message shows that your installation appears to be working correctly.`

### Step 2 — Image commands

```bash
docker pull nginx
docker images
```

### Step 3 — Container commands

Open **http://localhost:8888** after the `run` command to see the nginx welcome page.

```bash
docker run -d -p 8888:80 --name web nginx
docker ps
docker logs web
docker exec -it web bash
```

You are now inside the container. Type `ls` and `nginx -v`, then `exit`. Continue:

```bash
docker stop web
docker ps -a
docker start web
docker inspect web --format '{{.State.Status}}'
docker stop web
docker rm web
docker system df
```

### Step 4 — The Dockerfile (Practical 7)

You created this file in Practical 5. Show it:

```bash
cd ~/devops-lab
cat Dockerfile
```

```text
FROM python:3.12-slim          -> base image
WORKDIR /app                   -> working folder inside the image
COPY . .                       -> copy project files
RUN pip install -r requirements.txt
EXPOSE 5000                    -> port the app listens on
CMD ["python", "app.py"]        -> command run when the container starts
```

### Step 5 — Build the image and run it

```bash
docker build -t flask-app:v1 .
docker images flask-app
docker run -d -p 8000:5000 --name flask-container flask-app:v1
curl http://localhost:8000/health
```

Expected: `Server is up and running`. Open **http://localhost:8000** — the Container / Pod card shows the container ID.

```bash
docker logs flask-container
docker stop flask-container
docker rm flask-container
```

Optional — push to Docker Hub (replace the username):

```bash
docker login
docker tag flask-app:v1 your-dockerhub-username/flask-app:v1
docker push your-dockerhub-username/flask-app:v1
```

## Practical 8 — Multi-container app using Docker Compose

Two containers: a Flask web app and a Redis database that stores a visit counter. One command starts both.

### Step 1 — Create the project

```bash
mkdir -p ~/compose-demo && cd ~/compose-demo

cat > app.py <<'EOF'
from flask import Flask
import redis

app = Flask(__name__)
db = redis.Redis(host='redis', port=6379)


@app.route('/')
def home():
    count = db.incr('visits')
    return f'<h1>Hello from Docker Compose!</h1><h3>This page has been visited {count} times.</h3>'


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

cat > requirements.txt <<'EOF'
flask
redis
EOF

cat > Dockerfile <<'EOF'
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
EOF

cat > docker-compose.yml <<'EOF'
services:
  web:
    build: .
    ports:
      - "8000:5000"
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data

volumes:
  redis-data:
EOF
```

### Step 2 — Start the stack

```bash
docker compose up -d --build
docker compose ps
```

Expected: two containers `compose-demo-web-1` and `compose-demo-redis-1`, both `Up`. Open **http://localhost:8000** and refresh a few times — the counter goes up (it is stored in Redis).

```bash
docker compose logs web
```

### Step 3 — Show the data survives a restart

```bash
docker compose down
docker compose up -d
```

Refresh the page: the counter continues from the old number, because Redis data lives in the `redis-data` volume.

### Step 4 — Remove everything

```bash
docker compose down -v
```

## Practical 9 & 10 — Kubernetes: Pods, Deployments, Services, deploying the app

A local Kubernetes cluster runs inside Docker using **kind**. Practical 9 (Pods/Deployments/Services) and 10 (deploying your own app with a ConfigMap, scaling, rolling update, rollback) are done in one flow. It needs the `flask-app:v1` image from Practical 7.

### Step 1 — Install kind and kubectl

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.33.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

kind version
kubectl version --client
```

### Step 2 — Create the manifest files

```bash
mkdir -p ~/devops-lab/k8s && cd ~/devops-lab/k8s

cat > kind-config.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 30080
EOF

cat > configmap.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: flask-config
data:
  MESSAGE: "Hello from Kubernetes!"
EOF

cat > deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
        - name: flask-app
          image: flask-app:v1
          ports:
            - containerPort: 5000
          envFrom:
            - configMapRef:
                name: flask-config
EOF

cat > service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  type: NodePort
  selector:
    app: flask-app
  ports:
    - port: 80
      targetPort: 5000
      nodePort: 30080
EOF
```

### Step 3 — Create the cluster

```bash
kind create cluster --name devops-lab --config kind-config.yaml
kubectl wait --for=condition=Ready nodes --all --timeout=120s
kubectl cluster-info
kubectl get nodes
```

Expected: `devops-lab-control-plane   Ready   control-plane`.

### Step 4 — A single Pod (Practical 9)

```bash
kubectl run nginx-pod --image=nginx
kubectl get pods
kubectl describe pod nginx-pod
kubectl delete pod nginx-pod
```

Wait until `kubectl get pods` shows `Running` before `describe` (about 20 seconds).

### Step 5 — Deploy the Flask app (Deployment + Service + ConfigMap)

```bash
kind load docker-image flask-app:v1 --name devops-lab
kubectl apply -f configmap.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl rollout status deployment/flask-app
kubectl get all
```

Expected: 3 pods `Running`, `service/flask-service   NodePort   80:30080/TCP`. Wait about 10 seconds, then open **http://localhost:30080** — it says *Hello from Kubernetes!* (from the ConfigMap). Refresh a few times: the Pod name changes because the Service load-balances across 3 pods.

### Step 6 — Self-healing and scaling

Delete one pod; Kubernetes creates a new one immediately:

```bash
kubectl get pods
kubectl delete pod $(kubectl get pods -l app=flask-app -o name | head -1 | cut -d/ -f2)
kubectl get pods
```

```bash
kubectl scale deployment flask-app --replicas=5
kubectl get pods
```

### Step 7 — Rolling update to v2, then rollback

```bash
cd ~/devops-lab
sed -i "s/VERSION = 'v1'/VERSION = 'v2'/" app.py
docker build -t flask-app:v2 .
kind load docker-image flask-app:v2 --name devops-lab
kubectl set image deployment/flask-app flask-app=flask-app:v2
kubectl rollout status deployment/flask-app
```

Refresh **http://localhost:30080** → Version shows **v2**. Now roll back:

```bash
kubectl rollout history deployment/flask-app
kubectl rollout undo deployment/flask-app
kubectl rollout status deployment/flask-app
```

Refresh → Version is back to **v1**.

### Step 8 — Clean up

```bash
sed -i "s/VERSION = 'v2'/VERSION = 'v1'/" app.py
kind delete cluster --name devops-lab
```

## Practical 11 — Infrastructure provisioning using Terraform

No AWS account is needed: Terraform's Docker provider provisions an nginx server as a container on your machine, using the same init → plan → apply → destroy lifecycle as the manual's EC2 example.

### Step 1 — Install Terraform

```bash
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y terraform
terraform -version
```

### Step 2 — Write the configuration

```bash
mkdir -p ~/terraform-demo && cd ~/terraform-demo

cat > main.tf <<'EOF'
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}

provider "docker" {}

resource "docker_image" "nginx" {
  name = "nginx:latest"
}

resource "docker_container" "web" {
  name  = "terraform-nginx"
  image = docker_image.nginx.image_id

  ports {
    internal = 80
    external = 8081
  }
}

output "url" {
  value = "http://localhost:8081"
}
EOF
```

### Step 3 — init, validate, plan, apply

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```

Type `yes` when asked. Expected: `Plan: 2 to add, 0 to change, 0 to destroy.` then `Apply complete! Resources: 2 added` and `url = "http://localhost:8081"`. Open **http://localhost:8081** → *Welcome to nginx!*

### Step 4 — Inspect state and outputs

```bash
docker ps --filter name=terraform-nginx
terraform output
terraform state list
terraform show
```

### Step 5 — Destroy

```bash
terraform destroy
```

Type `yes`. Expected: `Destroy complete! Resources: 2 destroyed.`

## Practical 12 — Configuration management using Ansible

Two Ubuntu containers (`web1`, `web2`) act as the managed servers. Ansible installs nginx on both, copies a custom page and starts nginx — then a second run proves idempotency.

### Step 1 — Install Ansible and start two "servers"

```bash
sudo apt install -y ansible
ansible --version

docker run -d --name web1 -p 8091:80 ubuntu:24.04 sleep infinity
docker run -d --name web2 -p 8092:80 ubuntu:24.04 sleep infinity
```

### Step 2 — Inventory, page and playbook

```bash
mkdir -p ~/ansible-demo && cd ~/ansible-demo

cat > inventory.ini <<'EOF'
[webservers]
web1
web2

[webservers:vars]
ansible_connection=community.docker.docker
EOF

cat > index.html <<'EOF'
<h1>Configured by Ansible!</h1>
EOF

cat > playbook.yml <<'EOF'
- name: Install and configure Nginx
  hosts: webservers
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Copy custom index page
      copy:
        src: index.html
        dest: /var/www/html/index.html

    - name: Start nginx
      service:
        name: nginx
        state: started
EOF
```

### Step 3 — Prepare the servers and test connectivity

The fresh containers have no Python, so the `raw` module installs it first (takes about a minute):

```bash
ansible webservers -i inventory.ini -m raw -a "apt-get update && apt-get install -y python3"
ansible all -i inventory.ini -m ping
ansible webservers -i inventory.ini -a "hostname"
```

Expected: `web1 | SUCCESS => ... "ping": "pong"` and the same for web2.

### Step 4 — Run the playbook

```bash
ansible-playbook -i inventory.ini playbook.yml
```

Expected recap: `web1 : ok=4 changed=3 ... failed=0` (same for web2). Open **http://localhost:8091** and **http://localhost:8092** → *Configured by Ansible!*

### Step 5 — Idempotency (run it again)

```bash
ansible-playbook -i inventory.ini playbook.yml
```

Expected: `changed=0` for both hosts — nothing changes because the servers are already in the desired state.

Clean up: `docker rm -f web1 web2`

## Practical 13 — Monitoring using Prometheus and Grafana

Prometheus scrapes metrics from itself and from Node Exporter; Grafana shows them on a dashboard.

### Step 1 — Create the config files

```bash
mkdir -p ~/monitoring && cd ~/monitoring

cat > prometheus.yml <<'EOF'
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
EOF

cat > docker-compose.yml <<'EOF'
services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  node-exporter:
    image: prom/node-exporter
    ports:
      - "9100:9100"

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
EOF
```

### Step 2 — Start the stack

```bash
docker compose up -d
docker compose ps
```

### Step 3 — Prometheus (browser)

1. Open **http://localhost:9090/targets** → both `prometheus` and `node-exporter` show **UP**.
2. Go to **Query** and run these one by one (click **Execute**, then the **Graph** tab):

```text
up
node_cpu_seconds_total
rate(node_cpu_seconds_total[1m])
node_memory_MemAvailable_bytes
```

### Step 4 — Grafana (browser)

1. Open **http://localhost:3000** → login `admin` / `admin` (skip the password change).
2. **Connections → Data sources → Add data source → Prometheus** → URL `http://prometheus:9090` → **Save & test** → *Successfully queried the Prometheus API*.
3. **Dashboards → New → Import** → enter `1860` → **Load** → select the Prometheus data source → **Import**.

Expected: the *Node Exporter Full* dashboard with live CPU, memory, disk and network graphs.

### Step 5 — Stop

```bash
docker compose down
```

## Practicals skipped or combined

| Practical | What happened | Why |
| --- | --- | --- |
| 1 — Study of DevOps lifecycle | Skipped | Theory only, nothing to execute |
| 2 — Install Git/GitHub | Combined with 3 | Git is already installed; config + SSH are Steps 1–2 of Practical 3 |
| 6 — Docker basics | Combined with 7 | Docker Desktop is already installed; both use the same commands |
| 9 — Kubernetes intro | Combined with 10 | Same cluster and objects; 10 just deploys your own app |
| 11 — Terraform | Uses the Docker provider instead of AWS | No cloud account or cost needed; same init/plan/apply/destroy |
| 12 — Ansible | Uses 2 Ubuntu containers as servers | No VMs needed; same inventory, ping, playbook, idempotency |
| 14 — Mini project | Not included | Outside 1–13 |

## Troubleshooting

Every command in this guide was run end-to-end on your PC's Docker Desktop on 31 Oct 2026 (Ubuntu 24.04, Docker 29.8, kind v0.33.0, Terraform v1.17.1, Ansible core 2.16, Grafana 13.3) and gave the outputs shown. If something differs, check here first.

| Problem | Fix |
| --- | --- |
| `docker: command not found` in Ubuntu | Do setup Step 1 (Docker Desktop → WSL integration → Ubuntu), then close and reopen Ubuntu |
| `permission denied ... docker.sock` | In PowerShell run `wsl --shutdown`, then reopen Ubuntu |
| `port is already allocated` | `docker ps` to find the container using it, then `docker stop <name>` (port 5000 is your `studentdock-web`) |
| Terminal shows `>` and waits after pasting | The closing `EOF` line was not copied — type `EOF` and press Enter |
| `git push` asks for a password | The remote is HTTPS; use the SSH one: `git remote set-url origin git@github.com:<username>/devops-lab.git` |
| Jenkins console: `python3: not found` | The container was recreated — run the `docker exec -u root jenkins ...` line from Practical 4 again |
| http://localhost:30080 is blank right after deploy | Wait about 10 seconds and refresh |
| Kubernetes pods stuck `Pending` / PC slow | You have 3 old kind clusters running. Stop them before Practical 9 (command below) |
| Grafana import says no data source | Add the Prometheus data source first (Step 4.2), then import |

Stop your old kind clusters to free RAM (start them again later with `docker start` and the same names):

```bash
docker stop tutorial-cluster-control-plane tutorial-cluster-worker tutorial-cluster-worker2 tutorial-cluster-worker3 bankapp-tutorial-cluster-control-plane bankapp-tutorial-cluster-worker bankapp-tutorial-cluster-worker2 minishop-control-plane minishop-worker minishop-worker2
```
