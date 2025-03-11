# Creation of EKS Cluster

## Jenkins + Terraform + EKS

---

## Step 1: Launching an EC2 Instance and Installation of Required Tools

### 1.1 Launch VM - Ubuntu 22.04, t2.medium

### 1.2 Tools Installation

#### Java and Jenkins Installation Commands

```bash
#!/bin/bash
sudo apt update -y
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | sudo tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | sudo tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
/usr/bin/java --version
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl status jenkins
```

#### Terraform Installation

```bash
#!/bin/bash
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
wget -O- https://apt.releases.hashicorp.com/gpg | \
gpg --dearmor | \
sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
gpg --no-default-keyring \
--keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg \
--fingerprint
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update
sudo apt-get install terraform -y
```

#### K8S Installation

```bash
#!/bin/bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

#### AWS CLI Installation

```bash
#!/bin/bash
sudo apt install unzip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

---

## Step 2: Creation of IAM User and Generation of Secret Keys

Create IAM User - Generate Access and Secret Keys for Configuration.

---

## Step 3: Jenkins Setup and Configuration

- Install Pipeline Stage View plugin.
- Configure the access and secret keys in Jenkins.

---

## Step 4: Writing Terraform Files

Refer to the GitHub repository for the Terraform files.

**Repo URL:** [https://github.com/codexaman9901/Creation-of-EKS-Cluster-using-Terraform.git]

---

## Step 5: Creation of Jenkins Pipeline for Automating EKS Cluster Creation using Terraform

### Create Pipeline Job

#### Pipeline Script:

```groovy
pipeline {
    agent any
    environment {
        AWS_ACCESS_KEY_ID = credentials('AWS_Access_Key')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_Secret_Key')
        AWS_DEFAULT_REGION = 'ap-south-1'
    }
    stages{
        stage('Clone the Code'){
            steps{
                script{
                    checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/codexaman9901/Creation-of-EKS-Cluster-using-Terraform.git']])
                }
            }
        }
        stage('Terraform Initialization'){
            steps{
                script{
                    dir('terraform'){
                         sh 'terraform init'
                    }
                }
            }
        }
        stage('Terraform Validation'){
            steps{
                script{
                    dir('terraform'){
                         sh 'terraform validate'
                    }
                }
            }
        }
        stage('Infrastructure Checks'){
            steps{
                script{
                    dir('terraform'){
                         sh 'terraform plan'
                    }
                    input(message: "Approve?", ok: "proceed")
                }
            }
        }
        stage('Create/Destroy EKS cluster'){
            steps{
                script{
                    dir('terraform'){
                         sh 'terraform $action --auto-approve'
                    }
                }
            }
        }
    }
}
```

### Configuration Steps:

1. Before saving the job, go to **General** → Check **'This project is parameterized'**.
2. Add **Choice Parameter** → Name: **action** → Choices:
   - apply
   - destroy
3. Click **Apply** → **Save** → **Build With Parameters**.
4. Select **'apply'** → **Build**.

### Verification Steps:

- Once the job is built successfully, go to **EKS in AWS** → Verify under:
  - **Resources** tab
  - **Add-ons** tab
  - **Compute** tab

### Interacting with K8S Cluster:

```bash
aws eks update-kubeconfig --region us-east-1 --name <ClusterName>
kubectl run nginx --image=nginx
kubectl get pods
```

### Destroying the Cluster:

1. Open **Jenkins job**.
2. Click on **'Build with parameters'**.
3. Select **'Destroy'**.
4. Click **Build**.
5. Terminate the instance after destroying.

---

##
