# 🎮 Deploying Tetris with CI/CD, DevSecOps & ArgoCD (GitOps)

A complete DevSecOps pipeline that provisions infrastructure with **Terraform**, scans code and images with **SonarQube, OWASP Dependency-Check and Trivy**, builds and pushes a Docker image with **Jenkins**, and deploys to **AWS EKS** using **ArgoCD**.

![Architecture](architecture.gif)

---

## 📐 Architecture Diagram

```mermaid
flowchart LR
    DEV[App Code<br/>GitHub] --> JENKINS

    TF1[Terraform<br/>creates Jenkins EC2] -.-> JENKINS

    subgraph EC2[Jenkins Server - EC2]
        JENKINS[Jenkins]
    end

    JENKINS -->|Terraform| TF2[Terraform<br/>EKS + NodeGroup]
    TF2 --> EKS

    JENKINS --> SONAR[SonarQube<br/>Code Quality]
    SONAR --> NPM[npm install]
    NPM --> OWASP[OWASP<br/>Dependency Check]
    OWASP --> FSCAN[Trivy<br/>File Scan]
    FSCAN --> BUILD[Docker Build]
    BUILD --> HUB[(DockerHub<br/>Image Push)]
    HUB --> ISCAN[Trivy<br/>Image Scan]
    ISCAN --> MANIFEST[GitHub Manifest Repo<br/>Update image name]
    MANIFEST --> ARGO[ArgoCD]
    ARGO --> EKS

    subgraph EKS[Managed EKS Cluster]
        DEP[Deployment]
        SVC[Service / LoadBalancer]
    end

    SVC --> USER([Tetris Game in Browser])
```

### CI/CD flow (sequence)

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GH as GitHub (App Repo)
    participant J as Jenkins
    participant S as SonarQube
    participant T as Trivy
    participant DH as DockerHub
    participant M as GitHub (Manifest Repo)
    participant A as ArgoCD
    participant K as EKS Cluster

    Dev->>GH: Push code
    J->>GH: Checkout
    J->>S: Static code analysis + Quality Gate
    J->>J: npm install
    J->>T: Scan filesystem
    J->>DH: Build and push image
    J->>T: Scan image
    J->>M: Update image in deployment.yml
    A->>M: Detects change (auto sync)
    A->>K: Deploys new version
```

---

## 🗂️ Repositories

| Repo | Purpose |
|------|---------|
| `devsecops-Tetris-V1` | App code v1 + Terraform (`EKS_TERRAFORM`). Pipeline builds `tetrisv1` image and updates the manifest repo |
| `devsecops-tetris-v2` | App code v2. Pipeline builds `tetrisv2` image and updates the manifest repo |
| `devsecops-Tetris-manifest` | Kubernetes `deployment.yml` holding the latest image tag, watched by ArgoCD |

---

## 🧰 Tools Used

Docker · Jenkins · Java · SonarQube · AWS CLI · kubectl · Terraform · Trivy · OWASP Dependency-Check · ArgoCD · AWS EKS · DockerHub · GitHub

---

## ✅ Prerequisites

- AWS account
- EC2 instance: **m7i-flex.large**, Ubuntu 24 AMI, **50 GB** storage
- IAM role with Admin access attached to the instance
- DockerHub account
- GitHub account and a Personal Access Token (classic)
- S3 bucket for the Terraform backend

---

## Step 1: Setup the Jenkins EC2 Instance

### Script 1: Java, Jenkins, Docker

Save as `script1.sh`:

```bash
#!/bin/bash
set -e

# Java + tools
sudo apt update -y
sudo apt install -y openjdk-21-jdk wget curl
java -version

# Jenkins WAR
sudo mkdir -p /opt/jenkins
cd /opt/jenkins
[ -f jenkins.war ] || wget https://get.jenkins.io/war/2.552/jenkins.war
sudo chmod 755 jenkins.war
nohup java -Djava.net.preferIPv4Stack=true -jar /opt/jenkins/jenkins.war \
  > /opt/jenkins/jenkins.log 2>&1 &
sleep 5
ps -ef | grep jenkins.war | grep -v grep

# Docker
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo usermod -aG docker ubuntu
newgrp docker
```

Jenkins: `http://<EC2-IP>:8080` (log: `/opt/jenkins/jenkins.log`)

### Script 2: Terraform, kubectl, AWS CLI, Trivy

Save as `script2.sh`:

```bash
#!/bin/bash
# Terraform
sudo apt install wget -y
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# kubectl
sudo apt install curl -y
curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client

# AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt-get install unzip -y
unzip awscliv2.zip
sudo ./aws/install

# Trivy
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
sudo apt install -y libatomic1
```

### Run SonarQube

```bash
sudo chmod 777 /var/run/docker.sock
sudo systemctl daemon-reload
sudo systemctl restart docker.service

docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

SonarQube: `http://<EC2-IP>:9000` (default login `admin` / `admin`, change it on first login)

If Jenkins cannot talk to Docker, run:

```bash
sudo systemctl edit docker.service
# add:
# [Service]
# Environment=DOCKER_MIN_API_VERSION=1.24
sudo systemctl restart docker
docker start sonar
```

---

## Step 2: Jenkins Job 1 – Provision EKS with Terraform

1. Update the S3 bucket name in `devsecops-Tetris-V1/EKS_TERRAFORM/backend.tf`.
2. Install the **Terraform** plugin.
3. On the EC2 instance run `which terraform`.
4. **Manage Jenkins → Tools → Terraform**: Name `terraform`, uncheck *Install automatically*, install directory `/usr/bin/`.
5. **New Item → Pipeline → `EKSJOB`**, tick *This project is parameterized* → Choice parameter `action` with choices `apply` and `destroy`.

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout from Git') {
            steps { git branch: 'main', url: 'https://github.com/<your-user>/devsecops-Tetris-V1.git' }
        }
        stage('Terraform version') {
            steps { sh 'terraform --version' }
        }
        stage('Terraform init') {
            steps { dir('EKS_TERRAFORM') { sh 'terraform init' } }
        }
        stage('Terraform validate') {
            steps { dir('EKS_TERRAFORM') { sh 'terraform validate' } }
        }
        stage('Terraform plan') {
            steps { dir('EKS_TERRAFORM') { sh 'terraform plan' } }
        }
        stage('Terraform apply/destroy') {
            steps { dir('EKS_TERRAFORM') { sh 'terraform ${action} --auto-approve' } }
        }
    }
}
```

---

## Step 3: Jenkins Job 2 – Tetris CI Pipeline

### 3.1 Install plugins

**Manage Jenkins → Plugins → Available plugins**

- Eclipse Temurin installer
- SonarQube Scanner
- NodeJS
- OWASP Dependency-Check
- Docker, Docker Commons, Docker Pipeline, Docker API, Docker-build-step

### 3.2 Configure tools

**Manage Jenkins → Tools**

| Tool | Name | Installer |
|------|------|-----------|
| JDK | `jdk25` | Install from adoptium.net |
| SonarQube Scanner | `sonar-scanner` | Install automatically (latest) |
| NodeJS | `node25` | Install from nodejs.org (v25) |
| Dependency-Check | `DP-Check` | GitHub (latest) |
| Docker | `docker` | docker.com (latest) |

### 3.3 Connect SonarQube

1. SonarQube → **Administration → Security → Users → Tokens** → generate token named `TetrisVersion1.0`.
2. Jenkins → **Credentials → Global → Add**: Kind `Secret text`, ID `sonar-token`, secret = your token.
3. **Manage Jenkins → System → SonarQube servers**: Name `sonar-server`, URL `http://<EC2-IP>:9000`, token `sonar-token`.
4. SonarQube → **Administration → Configuration → Webhooks → Create**: Name `Jenkins`, URL `http://<EC2-IP>:8080/sonarqube-webhook/`, secret empty.

### 3.4 Add credentials

| ID | Kind | Value |
|----|------|-------|
| `docker` | Username with password | DockerHub username and password/token |
| `github` | Secret text | GitHub Personal Access Token (scopes: `repo`, `write:packages`, `admin:repo`) |
| `k8s` | Secret file | kubeconfig file |

> ⚠️ **Never commit tokens or passwords to Git or documentation.** Store them only in Jenkins Credentials.

### 3.5 Pipeline: `TETRISV1`

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk25'
        nodejs 'node25'
    }
    environment {
        SCANNER_HOME  = tool 'sonar-scanner'
        GIT_REPO_NAME = "devsecops-Tetris-manifest"
        GIT_USER_NAME = "<your-github-user>"
    }
    stages {
        stage('Clean Workspace') { steps { cleanWs() } }

        stage('Checkout from Git') {
            steps { git branch: 'main', url: 'https://github.com/<your-github-user>/devsecops-Tetris-V1.git' }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner \
                           -Dsonar.projectName=TetrisVersion1.0 \
                           -Dsonar.projectKey=TetrisVersion1.0 '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script { waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' }
            }
        }
        stage('Install Dependencies') { steps { sh 'npm install' } }

        stage('Trivy FS Scan') { steps { sh 'trivy fs . > trivyfs.txt' } }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh 'docker build -t tetrisv1 .'
                        sh 'docker tag tetrisv1 <dockerhub-user>/tetrisv1:latest'
                        sh 'docker push <dockerhub-user>/tetrisv1:latest'
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
            steps { sh 'trivy image <dockerhub-user>/tetrisv1:latest > trivyimage.txt' }
        }
        stage('Checkout Manifest Repo') {
            steps { git branch: 'main', url: 'https://github.com/<your-github-user>/devsecops-Tetris-manifest.git' }
        }
        stage('Update Deployment File') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                        NEW_IMAGE_NAME = "<dockerhub-user>/tetrisv1:latest"
                        sh "sed -i 's|image: .*|image: $NEW_IMAGE_NAME|' deployment.yml"
                        sh "git config --global user.email '<your-email>'"
                        sh "git config --global user.name '<your-github-user>'"
                        sh 'git add deployment.yml'
                        sh "git commit -m 'Update deployment image to $NEW_IMAGE_NAME'"
                        sh "git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main"
                    }
                }
            }
        }
    }
}
```

Click **Apply → Save → Build Now**. Then check the manifest repo: Jenkins will have committed the updated image in `deployment.yml`.

> 📄 Trivy reports are stored in the Jenkins workspace. Find them with `find / -name trivyimage.txt`.

---

## Step 4: Connect to the EKS Cluster

```bash
aws eks update-kubeconfig --name EKS_CLUSTER --region ap-south-1
kubectl get nodes
cat /root/.kube/config      # save as a secret file for the 'k8s' credential
```

Install the Kubernetes plugins: **Kubernetes Credentials, Kubernetes Client API, Kubernetes, Kubernetes CLI**.

Add credential: Kind `Secret file`, ID `k8s`.

---

## Step 5: Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml

# Expose argocd-server through a LoadBalancer
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Wait ~2 minutes, then get the address
sudo apt install jq -y
kubectl get svc -n argocd
export ARGOCD_SERVER=$(kubectl get svc argocd-server -n argocd -o json | jq --raw-output '.status.loadBalancer.ingress[0].hostname')
echo $ARGOCD_SERVER

# Initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Open the LoadBalancer URL and log in with username `admin` and the password above.

### Connect the manifest repo

**Settings (gear icon) → Repositories → Connect Repo Using HTTPS**

- Type: `git`
- Project: `default`
- URL: `https://github.com/<your-user>/devsecops-Tetris-manifest.git`

### Create the application

**Manage your applications → New App**

| Field | Value |
|-------|-------|
| Application Name | `tetris` |
| Project | `default` |
| Sync Policy | `Automatic` |
| Repository URL | manifest repo URL |
| Revision | `HEAD` |
| Path | `./` |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `default` |

Click **Create**. ArgoCD deploys the app and creates an AWS LoadBalancer for the `tetris-service`. Open the service hostname in your browser to play 🎮

```bash
kubectl get all
```

---

## Step 6: Upgrade to Version 2.0

Create a separate pipeline (`TETRISV2`). When new code lands in the v2 repo, the pipeline builds a new image, pushes it to DockerHub, updates `deployment.yml` in the manifest repo, and ArgoCD syncs it automatically.

Differences from V1:

- Checks out the **v2 app repo**
- Sonar project name/key: `TetrisVersion2.0`
- Image: `<dockerhub-user>/tetrisv2`, tagged with `${BUILD_NUMBER}` and `latest`
- Manifest repo checked out into a `manifest/` directory
- Commits only if `deployment.yml` actually changed
- 10-minute timeout on the Quality Gate
- `post` block prints success or failure message

```groovy
pipeline {
    agent any
    tools { jdk 'jdk25'; nodejs 'node25' }
    environment {
        SCANNER_HOME  = tool 'sonar-scanner'
        GIT_REPO_NAME = 'devsecops-Tetris-manifest'
        GIT_USER_NAME = '<your-github-user>'
        DOCKER_IMAGE  = '<dockerhub-user>/tetrisv2'
    }
    stages {
        stage('Clean Workspace') { steps { cleanWs() } }

        stage('Checkout from Git') {
            steps { git branch: 'main', url: 'https://github.com/<your-github-user>/devsecops-tetris-v2.git' }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=TetrisVersion2.0 \
                        -Dsonar.projectKey=TetrisVersion2.0
                    '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate(abortPipeline: false, credentialsId: 'sonar-token')
                }
            }
        }
        stage('Install Dependencies') { steps { sh 'npm install' } }
        stage('Trivy FS Scan')        { steps { sh 'trivy fs . > trivyfs.txt' } }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', url: 'https://index.docker.io/v1/', toolName: 'docker') {
                        sh '''
                            docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                            docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                            docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                            docker push ${DOCKER_IMAGE}:latest
                        '''
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
            steps { sh 'trivy image ${DOCKER_IMAGE}:${BUILD_NUMBER} > trivyimage.txt' }
        }
        stage('Checkout Manifest Repository') {
            steps {
                dir('manifest') {
                    git branch: 'main', url: 'https://github.com/<your-github-user>/devsecops-Tetris-manifest.git'
                }
            }
        }
        stage('Update Deployment File') {
            steps {
                dir('manifest') {
                    script {
                        withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                            sh '''
                                set -eu
                                NEW_IMAGE_NAME="${DOCKER_IMAGE}:${BUILD_NUMBER}"
                                sed -i "s|image: .*|image: ${NEW_IMAGE_NAME}|" deployment.yml

                                git config user.email '<your-email>'
                                git config user.name '<your-github-user>'
                                git add deployment.yml

                                if git diff --cached --quiet; then
                                    echo "No deployment changes to commit"
                                else
                                    git commit -m "Update deployment image to ${NEW_IMAGE_NAME}"
                                    git -c credential.helper= -c core.askPass=/bin/false \
                                        push "https://x-access-token:${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git" HEAD:main
                                fi
                            '''
                        }
                    }
                }
            }
        }
    }
    post {
        success { echo 'Tetris V2 DevSecOps Pipeline Completed Successfully!' }
        failure { echo 'Tetris V2 DevSecOps Pipeline Failed!' }
    }
}
```

After the build:

1. `deployment.yml` in the manifest repo now points to the v2 image.
2. ArgoCD shows the pods rolling out.
3. Refresh the LoadBalancer URL to see Version 2.

---

## 🧹 Cleanup

To avoid AWS charges, run the `EKSJOB` pipeline with `action = destroy`, then terminate the Jenkins EC2 instance.

---

## 🔐 Security Notes

- Rotate any token that has ever been pasted into notes or chat (SonarQube, GitHub, DockerHub).
- Change the default SonarQube `admin/admin` password.
- Avoid `chmod 777 /var/run/docker.sock` in production; add the `jenkins` user to the `docker` group instead.
- The IAM Admin role is for learning only; use least-privilege policies in real environments.
- Use immutable image tags (`${BUILD_NUMBER}`) rather than `latest` so ArgoCD detects every change.

---

## 🏁 Result

Every code push goes through quality analysis, dependency and vulnerability scanning, image build, and automated GitOps deployment to Kubernetes.
