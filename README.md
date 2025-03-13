# Deploying a DVWA clone on Kubernetes using DevSecOps methodology

In this project we would be deploying Netflix Clone application on an EKS cluster using DevSecOps methodology. We would be making use of security tools like SonarQube, OWASP Dependency Check and Trivy.
We would also be monitoring our EKS cluster using monitoring tools like Prometheus and Grafana. Most importantly we will be using ArgoCD for the Deployment.

## Step 1: Launch an EC2 Instance and install Jenkins, SonarQube, Docker and Trivy

We would be making use of Terraform to launch the EC2 instance. We would be adding a script as userdata for the installation of Jenkins, SonarQube, Trivy and Docker. 

## Step 2: Access Jenkins at port 8080 and install required plugins

Install the following plugins:

1. NodeJS 
2. Eclipse Temurin Installer
3. SonarQube Scanner
4. OWASP Dependency Check
5. Docker
6. Docker Commons
7. Docker Pipeline
8. Docker API
9. docker-build-step

Goto Jenkins Manage -> Script -> Run on console 

```bash
// jenkins manage -> script -> run on console 
import jenkins.model.*
import hudson.PluginManager
import hudson.model.UpdateCenter

def plugins = [
    "nodejs",
  	"adoptopenjdk",
    "sonar",
    "sonar-quality-gates",
    "quality-gates",
    "dependency-check-jenkins-plugin",
    "docker-plugin",
    "docker-commons",
    "docker-workflow",
    "docker-java-api",
    "docker-build-step",
    "docker-compose-build-step",
    "cloudbees-credentials",
    "aws-credentials",
    "pipeline-aws"
]

def instance = Jenkins.instance
def pm = instance.pluginManager
def uc = instance.updateCenter

plugins.each { plugin ->
    if (!pm.getPlugin(plugin)) {
        def updateSite = uc.getSites().first()
        def pluginInfo = updateSite.getPlugin(plugin)
        if (pluginInfo) {
            def installFuture = pluginInfo.deploy()
            installFuture.get()
            println "Installed plugin: ${plugin}"
        } else {
            println "Plugin not found in update center: ${plugin}"
        }
    } else {
        println "Plugin already installed: ${plugin}"
    }
}

instance.save()
println "Plugins installation completed. A Jenkins restart is recommended."
jenkins.reload()
```
## Step 3: Set up SonarQube

For the SonarQube Configuration, first access the Sonarqube Dashboard using the url http://elastic_ip:9000

1. Create the token 
Administration -> Security -> Users -> Create a token. Give name to the token - dvwa

2. Add this token as a credential in Jenkins 

3. Go to Manage Jenkins -> System -> SonarQube installation 
Give name to installation - sonar-sever. Add URL of SonarQube and for the credential select the one added in step 2. 

4. Go to Manage Jenkins -> Tools -> SonarQube Scanner Installations
-> Install automatically.

## Step 4: Set up OWASP Dependency Check 

1. Go to Manage Jenkins -> Tools -> Dependency-Check Installations
-> Install automatically

## Step 5: Set up Docker for Jenkins

1. Go to Manage Jenkins -> Tools -> Docker Installations -> Install automatically

2. And then go to Manage Jenkins -> Credentials -> System -> Global Credentials -> Add credentials. Add username and password for the docker registry (You need to create an account on Dockerhub). 

## Step 6: Create a pipeline in order to build and push the dockerized image securely using multiple security tools

Go to Dashboard -> New Item -> Pipeline 

Use the code below for the Jenkins pipeline. 

```bash
pipeline {
    agent any
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
                git branch: 'master', url: 'https://github.com/deepaksinghwrites/DVWA.git'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=dvwa \
                    -Dsonar.projectKey=dvwa'''
                }
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                echo "nothing to do here"
                // dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'OWASP DP-Check'
                // dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                script {
                    try {
                        sh "trivy fs . > trivyfs.txt" 
                    }catch(Exception e){
                        input(message: "Are you sure to proceed?", ok: "Proceed")
                    }
                }
            }
        }
        stage("Docker Build Image"){
            steps{
                
                sh "docker version"
                sh "docker-compose version"
                sh "docker-compose build --no-cache"
            }
        }
        stage("TRIVY"){
            steps{
                sh "docker images | grep dvwa"
                sh "trivy image --scanners vuln deepaksinghwrites/dvwa:latest"
                // Trivy may be defaulting to scanning the local filesystem instead of the Docker daemon. To explicitly use Docker mode use --scanners
                script{
                    input(message: "Are you sure to proceed?", ok: "Proceed")
                }
            }
        }
        stage("Docker Push"){
            steps{
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker'){   
                    sh "docker push deepaksinghwrites/dvwa:latest"
                    }
                }
            }
        }
    }
}
```

## Step 7: Create an EKS Cluster using Terraform 

Prerequisite: Install kubectl and helm before executing the commands below 

## Step 7.1: Create an IAM user and create access key and secret key 
ssh into the cicd server 
```bash
aws configure
```


## Step 8: Deploy Prometheus and Grafana on EKS 

In order to access the cluster use the command below:

```
aws eks update-kubeconfig --name "Cluster-Name" --region "Region-of-operation"
```

1. We need to add the Helm Stable Charts for your local.

```bash
helm repo add stable https://charts.helm.sh/stable
```

2. Add prometheus Helm repo

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

# auto complete the commands 
```bash
vi ~/.bashrc
```
```bash
source <(kubectl completion bash)
alias k=kubectl
complete -o default -F __start_kubectl k
```
```bash
source ~/.bashrc
```

3. Create Prometheus namespace

```bash
kubectl create namespace prometheus
```

4. Install kube-prometheus stack

```bash
helm install stable prometheus-community/kube-prometheus-stack -n prometheus
```

5. Edit the service and make it LoadBalancer

```
kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus
```

6. Edit the grafana service too to change it to LoadBalancer

```
kubectl edit svc stable-grafana -n prometheus
```

## Step 9: Deploy ArgoCD on EKS to fetch the manifest files to the cluster

1. Create a namespace argocd
```
kubectl create namespace argocd
```

2. Add argocd repo locally
```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml
```

3. By default, argocd-server is not publically exposed. In this scenario, we will use a Load Balancer to make it usable:
```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

4. We get the load balancer hostname using the command below:
```
kubectl get svc argocd-server -n argocd -o json
```

5. Once you get the load balancer hostname details, you can access the ArgoCD dashboard through it.

6. We need to enter the Username and Password for ArgoCD. The username will be admin by default. For the password, we need to run the command below:
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```









