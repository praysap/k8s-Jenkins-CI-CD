# 🚀 Kubernetes MERN Stack Deployment using Helm & Jenkins

This repository contains a complete Kubernetes deployment setup for a MERN (MongoDB, Express.js, React, Node.js) application using:
- 🧠 **Kubernetes(kubeadm)** for container orchestration
- 📦 **Helm** for templated, reusable Kubernetes manifests
- 🐳 **Docker** for containerizing the applications
- ⚙️ **Jenkins** (optional) for CI/CD automation
---

### 📁 Project Structure
```text
├── k8s/                                    # Raw Kubernetes YAML files (if any)
├── learnerReportCS_backend/                # Source code for backend (Express.js / Node.js)
├── learnerReportCS_frontend/               # Source code for frontend (React.js)
├── mern-chart/                             # Helm chart for deploying the entire MERN stack
│   ├── templates/                          # All Kubernetes manifests used by Helm
│   │   ├── backend.yaml                    # Backend Deployment & Service
│   │   ├── frontend.yaml                   # Frontend Deployment & Service
│   │   ├── mongo.yaml                      # MongoDB Deployment & Service
│   │   ├── secrets.yaml                    # Secret manifest for sensitive env variables ✅
│   ├── Chart.yaml                          # Helm chart metadata file
│   └── values.yaml                         # Configurable values (image, ports, env, etc.)
├── jenkinsfile                             # CI/CD pipeline for Jenkins
└── README.md                               # Project documentation (you are here!)
```
---
##  Setup Kubernetes [Kubeadm] Cluster (Version: 1.29)

### On both master & worker nodes
 <i>  Become root user </i>
```bash
sudo su
```

- <i>  Updating System Packages </i>
```bash
sudo apt-get update
```


 <i> Create a shell script 1.sh and paste the below code and run it :
```bash
#!/bin/bash
# disable swap
sudo swapoff -a

# Create the .conf file to load the modules at bootup
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

## Install CRIO Runtime
sudo apt-get update -y
sudo apt-get install -y software-properties-common curl apt-transport-https ca-certificates gpg

sudo curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/ /" | sudo tee /etc/apt/sources.list.d/cri-o.list

sudo apt-get update -y
sudo apt-get install -y cri-o

sudo systemctl daemon-reload
sudo systemctl enable crio --now
sudo systemctl start crio.service

echo "CRI runtime installed successfully"

# Add Kubernetes APT repository and install required packages
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet="1.29.0-*" kubectl="1.29.0-*" kubeadm="1.29.0-*"
sudo apt-get update -y
sudo apt-get install -y jq

sudo systemctl enable --now kubelet
sudo systemctl start kubelet
```

### On Master node
 <i> Create a shell script 2.sh and paste the below code and run it </i>
```bash
sudo kubeadm config images pull

sudo kubeadm init

mkdir -p "$HOME"/.kube
sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config


# Network Plugin = calico
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml

kubeadm token create --print-join-command
```

### On Worker node
 <i> Paste the join command you got from the master node and append --v=5 at the end </i>

```bash
<join-command> --v=5
```

- <i> Installing Docker </i>
```bash
sudo apt install docker.io -y
```
```bash
sudo chmod 777 /var/run/docker.sock
```
 <i> Create kubernetes namespace :
```bash
kubectl create namespace mern
```
<img width="722" height="59" alt="image" src="https://github.com/user-attachments/assets/fe12b4cc-f40d-4fd2-9df4-a4609731152f" />


 <i> Update kubernetes config context : 
```bash
kubectl config set-context --current --namespace mern
```
<img width="926" height="30" alt="image" src="https://github.com/user-attachments/assets/0959764b-3855-4dfc-a28b-62c8eb6d1910" />


### Enable DNS resolution on kubernetes cluster :

- Check coredns pod in kube-system namespace and you will find <i> Both coredns pods are running on master node </i>

```bash
kubectl get pods -n kube-system -o wide | grep -i core
```
<img width="843" height="76" alt="image" src="https://github.com/user-attachments/assets/47d03409-d880-4695-937b-30160eab8eb3" />

- Above step will run coredns pod on worker node as well for DNS resolution

```bash
kubectl edit deploy coredns -n kube-system -o yaml
```
- <i> Make replica count from 2 to 4 </i>
<img width="847" height="527" alt="image" src="https://github.com/user-attachments/assets/39f91fdc-b840-40bb-be43-126e7683f6f4" />


 <i>  **Build the Docker image (frontend)**
```bash
docker image build --no-cache --build-arg REACT_APP_API_BASE_URL=http://10.228.12.107:30585 -t praysap/learner-frontend:latest .
```
 <i> **Push the image to Docker Hub**
```bash
docker push praysap/learner-frontend:latest
```
<img width="944" height="441" alt="image" src="https://github.com/user-attachments/assets/ef1191ce-0114-461f-a9bf-e604704851ef" />


 <i> **Build the Docker image (backend)**
```bash
docker build -t praysap/learner-backend:latest .
```
- <i> **Push the image to Docker Hub**
```bash
docker push praysap/learner-backend:latest
```
<img width="944" height="434" alt="image" src="https://github.com/user-attachments/assets/ea646965-0d78-47ac-9080-3c044fe3ef77" />
<img width="941" height="177" alt="image" src="https://github.com/user-attachments/assets/ecb46d95-6905-4f39-8f0c-846770bbfe25" />


 ### 📥 Install Helm CLI on Linux/mascOS
```bash
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```
### 📥 Install Helm on On Windows (via Chocolatey):
```bash
choco install kubernetes-helm
```
```bash
helm version
```
---
### 🛠 Create Helm Chart
```bash
helm create mern-app
```
### 🚀 Deployment with Helm
```bash
cd mern-app dir
```
```bash
kubectl config set-context --current --namespace mern
```
```bash
helm upgrade --install mern-app . --namespace mern --create-namespace
```
---
### ✅ Get running pods
```bash
kubectl get pods -n mern
```
<img width="939" height="450" alt="image" src="https://github.com/user-attachments/assets/e6ea82e5-7c00-41ef-af29-d77eb47b2374" />

---
### ✅ Verify Deployment
```bash
kubectl get all -n mern
```
- Make sure all pods are 1/1 READY and services are running (frontend-service, backend-service, mongo).
<img width="939" height="450" alt="image" src="https://github.com/user-attachments/assets/0b610ea9-9fa1-4b86-9e39-87bd0c1d027f" />
  
---
### 🔁 Access a running pod (e.g. frontend)
```bash
kubectl exec -it frontend-deployment-67f9fbc9f6-pt6wq -n mern -- /bin/sh
```
 <img width="939" height="450" alt="image" src="https://github.com/user-attachments/assets/69ecd42c-f198-4f1f-9d7c-9cfc71f7a2c6" />

---
### 🧪 Delete All Resources in Namespace
```bash
kubectl delete all --all -n mern
```
- This deletes:Pods,Services,Deployments,ReplicaSets and Any other workload in that namespace
---

#### Minikube [local]

<img width="939" height="238" alt="image" src="https://github.com/user-attachments/assets/7a4b26e1-69f7-47ad-a970-b5f6e741252d" />

#### Minikube [EC-2/Ubuntu]

<img width="939" height="254" alt="image" src="https://github.com/user-attachments/assets/6e02e259-0119-43c0-88d2-e2430cba0f1b" />


## 🌐 Access the App 
### Test via Port Forwarding [Local]
```bash
kubectl port-forward svc/frontend-service 8080:80 -n mern
kubectl port-forward svc/backend-service 3000:3000 -n mern
kubectl port-forward svc/mongo 28017:27017 -n mern
```
Frontend App available at: http://10.228.12.107:31063<br>

<img width="939" height="450" alt="image" src="https://github.com/user-attachments/assets/b99eabd2-45f7-47fa-b0bc-78533f55a6df" />
<img width="939" height="450" alt="image" src="https://github.com/user-attachments/assets/1d4cdccb-de77-482f-91f4-c692f5e98694" />
<img width="939" height="450" alt="image" src="https://github.com/user-attachments/assets/7262e7b8-9d4c-4b94-8f44-73686840677a" />

Backend App available at: http://10.228.12.107:30585<br>

<img width="939" height="450" alt="image" src="https://github.com/user-attachments/assets/f391ee13-f16d-48e8-a147-652a92828c19" />


Mongo DB available at: mongodb://localhost:27017<br>

### Test via Port Forwarding [Ec-2/Ubuntu]
```bash
kubectl port-forward svc/frontend-service -n mern 8080:80 --address=0.0.0.0
kubectl port-forward svc/backend-service -n mern 3000:3000 --address=0.0.0.0
kubectl port-forward svc/mongo 28017:27017 -n mern
```

Mongo DB available at: mongodb://public-ip:28017<br>

### Test via Minikube (OPTIONAL)
```bash
minikube service frontend-service -n mern
```
- This will open your browser or give you a URL — check the site.
---

## Configuration
### .env file in Backend
```bash
ATLAS_URI=mongodb://mongo:27017/blog_mern
```
### .env file in Frontend
Based on your setup you need to update.
```bash
REACT_APP_API_BASE_URL=http://backend-service:3000
OR
REACT_APP_API_BASE_URL=http://localhost:3000
```
---

## ⚙️ Jenkins configuration
### Pipeline setup
1. Create Jenkinsfile inside your project directory.
2. Create dockerhub credentials in jenkins.
3. Create jenkins pipeline.

```
pipeline {
    agent any
    
    environment {
        REPO_URL = 'https://github.com/vipulsaw/Container-Orchestration-2.git'
        REPO_NAME = 'Container-Orchestration-2'
        HELM_CHART_DIR = 'mern-app'
        NAMESPACE = 'mern'
        RELEASE_NAME = 'mern-app'
        EC2_INSTANCE_IP = '13.203.66.123'
        DOCKER_HUB_REPO = 'vipulsaw123'
    }
    
    stages {
        stage('Checkout Source Code') {
            steps {
                checkout([$class: 'GitSCM', 
                         branches: [[name: '*/main']], 
                         userRemoteConfigs: [[url: env.REPO_URL]]])
                echo "Repository cloned successfully"
            }
        }
        
        stage('Build Docker Images') {
            steps {
                script {
                    docker.build("${env.DOCKER_HUB_REPO}/learner-frontend:latest", './learnerReportCS_frontend')
                    docker.build("${env.DOCKER_HUB_REPO}/learner-backend:latest", './learnerReportCS_backend')
                    echo "Docker images built successfully"
                }
            }
        }
        
        stage('Push Docker Images') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-credentials-Vipul', 
                        usernameVariable: 'DOCKER_HUB_USER', 
                        passwordVariable: 'DOCKER_HUB_PASSWORD'
                    )]) {
                        sh "echo \$DOCKER_HUB_PASSWORD | docker login -u \$DOCKER_HUB_USER --password-stdin"
                        sh "docker push ${env.DOCKER_HUB_REPO}/learner-frontend:latest"
                        sh "docker push ${env.DOCKER_HUB_REPO}/learner-backend:latest"
                        echo "Docker images pushed to Docker Hub successfully"
                    }
                }
            }
        }
        
        stage('Copy Repository to Target EC2') {
            steps {
                sshagent(credentials: ['vipulroot']) {
                    script {
                        sh """
                            ssh -o StrictHostKeyChecking=no root@${env.EC2_INSTANCE_IP} \
                            'mkdir -p ~/${env.REPO_NAME}'
                        """
                        sh """
                            rsync -avz -e "ssh -o StrictHostKeyChecking=no" \
                            --exclude='.git' \
                            --delete \
                            ./ root@${env.EC2_INSTANCE_IP}:~/${env.REPO_NAME}/
                        """
                        echo "Files copied to EC2 instance successfully"
                    }
                }
            }
        }
        
        stage('Install/Upgrade Helm Release') {
            steps {
                sshagent(credentials: ['vipulroot']) {
                    script {
                        sh """
                            ssh -o StrictHostKeyChecking=no root@${env.EC2_INSTANCE_IP} \
                            'cd ~/${env.REPO_NAME}/${env.HELM_CHART_DIR} && \
                            helm upgrade --install ${env.RELEASE_NAME} . \
                            --namespace ${env.NAMESPACE} \
                            --create-namespace'
                        """
                        echo "Helm release upgraded successfully"
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                sshagent(credentials: ['vipulroot']) {
                    script {
                        sh """
                            ssh -o StrictHostKeyChecking=no root@${env.EC2_INSTANCE_IP} \
                            'helm list -n ${env.NAMESPACE} && \
                            kubectl get pods -n ${env.NAMESPACE}'
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed!'
        }
    }
}
```

<img width="959" height="440" alt="image" src="https://github.com/user-attachments/assets/c82945fa-2d64-45ce-8548-69a006d47d03" />


### 🛠 Debugging & Helm Commands for Kubernetes (Namespace: mern)

Use the following commands to debug and inspect your resources within the `mern` namespace.

### 🔍 Kubernetes Commands

  <i>  **Get all pods**
```bash
kubectl get pods -n mern
```
 <i> **Describe a specific pod**
```bash
kubectl describe pod <pod-name> -n mern
```
 <i> **View logs of a pod**
```bash
kubectl logs <pod-name> -n mern
```
  <i> **Follow logs in real-time**
```bash
kubectl logs -f <pod-name> -n mern
```

 ###  Exec into a pod using bash (or sh if bash is unavailable)
```bash
kubectl exec -it <pod-name> -n mern -- /bin/bash
```
```bash
kubectl exec -it <pod-name> -n mern -- /bin/sh
```
  <i>  **List services**
```bash
kubectl get svc -n mern
```

  <i>  **Describe a specific service**
```bash
kubectl describe svc learn-api-service -n mern
```
  <i>  **List deployments**
```bash
kubectl get deployments -n mern
```
  <i>  **Describe a deployment**
```bash
kubectl describe deployment learn-api -n mern
```
 <i> **Port forward to access service locally**
```bash
kubectl port-forward service/learn-api-service 3001:3001 -n mern
```
   <i>  **Test service locally via curl**
```bash
curl localhost:3001
```


### 📦 Helm Commands

  <i>  **List Helm releases in namespace**
```bash
helm list -n mern
```

  <i>  **Get Helm chart values**
```bash
helm get values learn-api -n mern
```

 <i> **Get history of Helm deployments**
```bash
helm history <chart-name>
```

  <i>  **View rendered Helm manifest** 
```bash
helm get manifest learn-api -n mern
```

