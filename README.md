# Netflix-Clone-Deployment---Devops

Architecture
![image](https://github.com/Skillz619/Netflix-Clone-Deployment---Devops/assets/43133388/ec0e6ddd-55ac-4fd9-9e5f-74a9a3615486)

## Phase 1: Initial Setup and Deployment

### Step 1: Launch EC2 (Ubuntu 22.04):

- Provision an EC2 instance on AWS with Ubuntu 22.04.
- Connect to the instance using SSH.

### Step 2: Clone the Code:

- Update all the packages and then clone the code.
- Clone your application’s code repository onto the EC2 instance:

```bash
https://github.com/Skillz619/Netflix-Clone-Deployment---Devops.git
```

### Step 3: Install Docker and Run the App Using a Container:

- Set up Docker on the EC2 instance:

```bash
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER # Replace with your system’s username, e.g., ‘ubuntu’
newgrp docker
sudo chmod 777 /var/run/docker.sock
```

- Build and run your application using Docker containers:

```bash
docker build -t netflix .
docker run -d — name netflix -p 8081:80 netflix:latest

# To delete
docker stop <containerid>
docker rmi -f netflix
```

(Note: You’ll encounter an error because you need an API key. We’ll address this in the next step.)

### Step 4: Get the API Key:

- Open a web browser and navigate to TMDB (The Movie Database) website.
- Click on “Login” and create an account.
- Once logged in, go to your profile and select “Settings.”
- Click on “API” from the left-side panel.
- Create a new API key by clicking “Create” and accepting the terms and conditions.
- Provide the required basic details and click “Submit.”
- You will receive your TMDB API key.

Now, recreate the Docker image with your API key:

```bash
docker build — build-arg TMDB_V3_API_KEY=<your-api-key> -t netflix .
```

## Phase 2: Security

### Step 1: Install SonarQube and Trivy:

- Install SonarQube and Trivy on the EC2 instance to scan for vulnerabilities.

To run SonarQube:

```bash
docker run -d — name sonar -p 9000:9000 sonarqube:lts-community
```

Access it at `publicIP:9000` (default username & password is admin).

To install Trivy:

```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO — https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

To scan an image using Trivy:

```bash
trivy image <imageid>
```

### Step 2: Integrate SonarQube and Configure:

- Integrate SonarQube with your CI/CD pipeline.
- Configure SonarQube to analyze code for quality and security issues.

## Phase 3: CI/CD Setup

### Step 1: Install Jenkins for Automation:

```bash
sudo apt update
sudo apt install fontconfig openjdk-17-jre
java -version
# (Ensure Java 17 is installed)

# Jenkins
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
/etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

Access Jenkins at `publicIp:8080`.

### Step 2: Install Necessary Plugins in Jenkins:

- Eclipse Temurin Installer (Install without restart)
- SonarQube Scanner (Install without restart)
- NodeJs Plugin (Install Without restart)
- Email Extension Plugin

### Step 3: Configure Java and Nodejs in Global Tool Configuration:

- Go to “Manage Jenkins” → “Tools” → Install JDK(17) and NodeJs(16)→ Click on Apply and Save.

### Step 4: Create a Jenkins webhook.

pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/saicharan20/DevSecOps-Project.git'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix'''
                }
            }
        }
        stage("quality gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
    }
}
Install Dependency-Check and Docker Tools in Jenkins

Install Dependency-Check Plugin:

Go to “Dashboard” in your Jenkins web interface.
Navigate to “Manage Jenkins” → “Manage Plugins.”
Click on the “Available” tab and search for “OWASP Dependency-Check.”
Check the checkbox for “OWASP Dependency-Check” and click on the “Install without restart” button.
Configure Dependency-Check Tool:

After installing the Dependency-Check plugin, you need to configure the tool.
Go to “Dashboard” → “Manage Jenkins” → “Global Tool Configuration.”
Find the section for “OWASP Dependency-Check.”
Add the tool’s name, e.g., “DP-Check.”
Save your settings.
Install Docker Tools and Docker Plugins:

Go to “Dashboard” in your Jenkins web interface.
Navigate to “Manage Jenkins” → “Manage Plugins.”
Click on the “Available” tab and search for “Docker.”
Check the following Docker-related plugins:
Docker
Docker Commons
Docker Pipeline
Docker API
docker-build-step

pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/saicharan20/DevSecOps-Project.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build --build-arg TMDB_V3_API_KEY=<yourapikey> -t netflix ."
                       sh "docker tag netflix saicharan20/netflix:latest "
                       sh "docker push saicharan20/netflix:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image saicharan20/netflix:latest > trivyimage.txt" 
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name netflix -p 8081:80 saicharan20/netflix:latest'
            }
        }
    }
}


If you get docker login failed errorr

sudo su
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

Click on the “Install without restart” button to install these plugins.

Add DockerHub Credentials:

To securely handle DockerHub credentials in your Jenkins pipeline, follow these steps:
Go to “Dashboard” → “Manage Jenkins” → “Manage Credentials.”
Click on “System” and then “Global credentials (unrestricted).”
Click on “Add Credentials” on the left side.
Choose “Secret text” as the kind of credentials.
Enter your DockerHub credentials (Username and Password) and give the credentials an ID (e.g., “docker”).
Click “OK” to save your DockerHub credentials.

## Phase 4: Monitoring

Install Prometheus and Grafana:

Set up Prometheus and Grafana to monitor your application.

Installing Prometheus:

First, create a dedicated Linux user for Prometheus and download Prometheus

### Step 1: Install Prometheus and Grafana:

Set up Prometheus and Grafana to monitor your application.
sudo useradd --system --no-create-home --shell /bin/false prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz

tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
cd prometheus-2.47.1.linux-amd64/
sudo mkdir -p /data /etc/prometheus
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml

sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
sudo nano /etc/systemd/system/prometheus.service
Add the following content to the prometheus.service file:

[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
Enable and start Prometheus:

sudo systemctl enable prometheus
sudo systemctl start prometheus

sudo systemctl status prometheus
Installing Node Exporter:

Create a system user for Node Exporter and download Node Exporter:

sudo useradd --system --no-create-home --shell /bin/false node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz

tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter*
Create a systemd unit configuration file for Node Exporter:

sudo nano /etc/systemd/system/node_exporter.service

[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
Replace --collector.logind with any additional flags as needed.

sudo systemctl enable node_exporter
sudo systemctl start node_exporter

sudo systemctl status node_exporter
Enable and start Node Exporter:

### Step 2: Configure Prometheus Plugin Integration:

Integrate Jenkins with Prometheus to monitor the CI/CD pipeline.
Prometheus Configuration:
To configure Prometheus to scrape metrics from Node Exporter and Jenkins, you need to modify the prometheus.yml file. Here is an example prometheus.yml configuration for your setup:
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<your-jenkins-ip>:<your-jenkins-port>']

promtool check config /etc/prometheus/prometheus.yml

curl -X POST http://localhost:9090/-/reload
Install Grafana on Ubuntu 22.04 and Set it up to work with Prometheus

Step 1: Install Dependencies:

First, ensure that all necessary dependencies are installed:

sudo apt-get update
sudo apt-get install -y apt-transport-https software-properties-common

wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -

echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get -y install grafana

sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server

## Phase 6: Kubernetes

Monitor Kubernetes with Prometheus

Prometheus is a powerful monitoring and alerting toolkit, and you’ll use it to monitor your Kubernetes cluster. Additionally, you’ll install the node exporter using Helm to collect metrics from your cluster nodes.

Install Node Exporter using Helm

To begin monitoring your Kubernetes cluster, you’ll install the Prometheus Node Exporter. This component allows you to collect system-level metrics from your cluster nodes. Here are the steps to install the Node Exporter using Helm:

Add the Prometheus Community Helm repository:
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
2. Create a Kubernetes namespace for the Node Exporter:

kubectl create namespace prometheus-node-exporter
3. Install the Node Exporter using Helm:

helm install prometheus-node-exporter prometheus-community/prometheus-node-exporter --namespace prometheus-node-exporter
Update your Prometheus configuration (prometheus.yml) to add a new job for scraping metrics from nodeip:9001/metrics. You can do this by adding the following configuration to your prometheus.yml file:

  - job_name: 'Netflix'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['node1Ip:9100']
Deploy Application with ArgoCD
Install ArgoCD:
You can install ArgoCD on your Kubernetes cluster by following the instructions provided in the EKS Workshop documentation.
Set Your GitHub Repository as a Source:
After installing ArgoCD, you need to set up your GitHub repository as a source for your application deployment. This typically involves configuring the connection to your repository and defining the source for your ArgoCD application. The specific steps will depend on your setup and requirements.

Cleanup Everything
Terminate all the created resources


By following these steps, you’ll have a fully functional DevSecOps pipeline for deploying and managing a Netflix Clone on the cloud. Remember to adapt the instructions to your specific environment and requirements.
