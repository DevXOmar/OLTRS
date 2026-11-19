# OLTRS — Software Engineering Lab Internal II (SET-8)

A practical, copy-paste guide for **Windows (exam machine)** and **macOS (home practice)**. Repository: **https://github.com/DevXOmar/OLTRS**.

**Do the questions in this order:** **Q2 Jenkins → Q1 Docker Compose + Nagios → Q3 Minikube → Q4 AWS EC2**. Q2 creates the Docker Hub image reused in Q1 and Q3. This guide is for the KMIT SET-8 lab, not a production deployment.

> Only two values must be supplied by each student: **Docker Hub username** (the image will be <code>YOUR_DOCKERHUB_USERNAME/oltrs:latest</code>) and **private login credentials**, configured locally. Never commit passwords, Docker Hub tokens, ngrok tokens, Gmail app passwords or AWS <code>.pem</code> keys. GitHub repository and branch are fixed to <code>DevXOmar/OLTRS</code> and <code>main</code>.

## 0. Start tools and check your machine

Required: [JDK 21](https://adoptium.net/), [Maven](https://maven.apache.org/download.cgi), [Git](https://git-scm.com/downloads), [Jenkins LTS](https://www.jenkins.io/download/), [Docker Desktop](https://www.docker.com/products/docker-desktop/), [ngrok](https://ngrok.com/download), [Minikube](https://minikube.sigs.k8s.io/docs/start/) and [kubectl](https://kubernetes.io/docs/tasks/tools/). **Docker Compose is bundled with Docker Desktop. Tomcat and Nagios will run in Docker; no separate installations are needed.**

### Windows — PowerShell (priority)

Start Docker Desktop from the Start menu (or):

~~~powershell
Start-Process "C:\Program Files\Docker\Docker\Docker Desktop.exe"
# Run PowerShell as Administrator for these service commands:
Get-Service *jenkins*
Start-Service Jenkins
Get-Service Jenkins
Start-Process "http://localhost:8080"
~~~

If Jenkins is already running, <code>Start-Service</code> may report that it is running; that's fine. If the service has another name, use the name returned by <code>Get-Service *jenkins*</code>.

Check tools (ordinary PowerShell):

~~~powershell
java -version
mvn -version
git --version
docker version
docker compose version
docker buildx version
minikube version
kubectl version --client
ngrok version
~~~

### macOS — Terminal

~~~bash
brew services start jenkins-lts
brew services list
open -a Docker
open http://localhost:8080

java -version
mvn -version
git --version
docker version
docker compose version
docker buildx version
minikube version
kubectl version --client
ngrok version
~~~

Wait until Docker Desktop says its engine is running before using Docker commands.

### Ports used throughout the lab

| Tool | Host address | Container port / notes |
| --- | --- | --- |
| Jenkins GUI | http://localhost:8080 | Jenkins host service |
| Tomcat deployment | http://localhost:8085 | 8085 → Tomcat 8080 |
| OLTRS via Compose | http://localhost:8088 | 8088 → app 8080 |
| Nagios | http://localhost:8888/nagios/ | 8888 → Nagios 80 |
| Minikube port-forward | http://localhost:8081 | 8081 → app 8080 |
| AWS EC2 | http://PUBLIC_IP:8080 | EC2 inbound TCP 8080 |

**Find occupied ports:**

~~~powershell
# Windows — listeners on the lab ports
Get-NetTCPConnection -State Listen | Where-Object { $_.LocalPort -in 8080,8081,8085,8088,8888 } | Format-Table LocalAddress,LocalPort,OwningProcess
# Identify a Windows process from its PID, e.g.:
Get-Process -Id 1234
# Docker's published ports (both systems):
docker ps --format "table {{.Names}}\t{{.Ports}}"
~~~

~~~bash
# macOS — listening processes on lab ports
lsof -nP -iTCP -sTCP:LISTEN | grep -E ':(8080|8081|8085|8088|8888) '
# Docker's published ports
docker ps --format "table {{.Names}}\t{{.Ports}}"
~~~

---

## Q2 (25M) — Jenkins CI/CD: Git → Maven → WAR → Tomcat → Docker Hub

### 1. Get this repository

On either Windows PowerShell or Mac Terminal:

~~~bash
git clone https://github.com/DevXOmar/OLTRS.git
cd OLTRS
git branch --show-current
git status
~~~

Branch should be **main**. This repo already contains the correct <code>Dockerfile</code>; don't overwrite it with a blank file.

### 2. Dockerfile (already present — reference copy)

The exact multi-stage <code>Dockerfile</code> in the repository:

~~~dockerfile
# Build the Maven web application
FROM maven:3.9-eclipse-temurin-21 AS build

WORKDIR /app

COPY pom.xml .
COPY src ./src

RUN mvn -B -ntp clean package

# Run the WAR using Tomcat 9
FROM tomcat:9.0-jre17-temurin-noble

COPY --from=build /app/target/ticket-reservation.war /usr/local/tomcat/webapps/ROOT.war

EXPOSE 8080

CMD ["catalina.sh", "run"]
~~~

The Maven <code>pom.xml</code> packages a WAR called <code>target/ticket-reservation.war</code>. Tomcat 9 runs it at the root context (<code>/</code>).

### 3. Start Tomcat — same commands on Windows and Mac

Run in **PowerShell or Terminal** (with Docker Desktop running):

~~~bash
docker pull tomcat:9.0-jre17-temurin-noble
docker run -d --name tomcat-local -p 8085:8080 tomcat:9.0-jre17-temurin-noble
docker ps
~~~

Open http://localhost:8085 **after the Jenkins WAR deployment**. If the named container already exists, do **not** run <code>docker run</code> again; use:

~~~bash
docker start tomcat-local
docker logs tomcat-local
~~~

For an intentional clean reset only: <code>docker rm -f tomcat-local</code>, then repeat the pull/run commands.

### 4. Prepare Jenkins GUI and Docker Hub credentials

Open **http://localhost:8080**. Verify plugins: **Pipeline, Git, GitHub, Credentials Binding**. Under **Manage Jenkins → Tools**, make sure Git, JDK and Maven are configured or available in Jenkins's PATH.

In Docker Hub, create a repository named **oltrs** under your own username.

In **Jenkins → Manage Jenkins → Credentials → System → Global credentials → Add Credentials**:

| Field | Value |
| --- | --- |
| Kind | Username with password |
| Username | Your Docker Hub username |
| Password | Docker Hub access token (not your account password) |
| ID | **dockerhub-creds** |

The credential ID must match the Pipeline script. Jenkins must have access to Docker Desktop; Windows service accounts often need explicit Docker access (see troubleshooting below).

### 5. Jenkins Scripted Pipeline — copy all of this into Jenkins

**New Item → Pipeline → name <code>OLTRS-Pipeline</code> → OK → Pipeline → Definition: Pipeline script → paste → Save → Build Now → Console Output.**

This is the **working version used in practice**, with the macOS Maven PATH fix and without the troublesome email step. The Windows branch runs <code>bat</code> and <code>powershell</code> commands.

~~~groovy
node {
    // OLTRS uses the main branch.
    def repo = 'https://github.com/DevXOmar/OLTRS.git'

    // Homebrew path for Apple Silicon Macs (plus tested Maven path).
    if (isUnix()) {
        env.PATH = "/usr/local/bin:/opt/maven/apache-maven-3.9.16/bin:/opt/homebrew/bin:${env.PATH}"
    }

    try {
        stage('1 - Git Checkout') {
            deleteDir()
            git branch: 'main', url: repo
        }

        stage('2 - Maven Build and Test') {
            if (isUnix()) {
                sh 'mvn -B clean install'
            } else {
                bat 'mvn -B clean install'
            }
        }

        stage('3 - Verify WAR') {
            if (isUnix()) {
                sh 'test -f target/ticket-reservation.war'
            } else {
                bat 'if not exist target\\ticket-reservation.war exit /b 1'
            }
        }

        stage('4 - Deploy to Tomcat') {
            if (isUnix()) {
                sh 'docker cp target/ticket-reservation.war tomcat-local:/usr/local/tomcat/webapps/ROOT.war'
            } else {
                bat 'docker cp target\\ticket-reservation.war tomcat-local:/usr/local/tomcat/webapps/ROOT.war'
            }
        }

        stage('5 - Docker Build and Push') {
            withCredentials([
                usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_TOKEN'
                )
            ]) {
                if (isUnix()) {
                    sh '''
                        set -eu
                        printf '%s' "$DH_TOKEN" | docker login --username "$DH_USER" --password-stdin
                        docker buildx build --platform linux/amd64,linux/arm64 -t "$DH_USER/oltrs:latest" --push .
                        docker logout
                    '''
                } else {
                    powershell '''
                        $ErrorActionPreference = "Stop"

                        $env:DH_TOKEN | docker login --username $env:DH_USER --password-stdin
                        if ($LASTEXITCODE -ne 0) { throw "Docker login failed" }

                        docker buildx build --platform linux/amd64,linux/arm64 -t "$env:DH_USER/oltrs:latest" --push .
                        if ($LASTEXITCODE -ne 0) { throw "Docker build/push failed" }

                        docker logout
                    '''
                }
            }
        }

        currentBuild.result = 'SUCCESS'
    }
    catch (err) {
        currentBuild.result = 'FAILURE'
        echo "BUILD FAILED: ${err.getMessage()}"
        throw err
    }
}
~~~

**Expected:** green stages, WAR successfully copied to the running <code>tomcat-local</code> container, OLTRS accessible at http://localhost:8085, and **oltrs:latest** published in your Docker Hub repository.

**If Buildx cannot build both platforms:** run the following in the same Docker environment used by the Jenkins agent, then retry:

~~~bash
docker buildx create --name labbuilder --driver docker-container --use
docker buildx inspect --bootstrap
~~~

If <code>labbuilder</code> already exists, use <code>docker buildx use labbuilder</code>. On Windows, Jenkins must run under a user/account with access to that Docker context and builder.

### 6. Locate Java/Maven/Git/Docker paths when Jenkins cannot find them

**Windows — PowerShell (use this first):**

~~~powershell
where.exe java
where.exe mvn
where.exe git
where.exe docker
Get-Command java,mvn,git,docker | Select-Object Name,Source
$env:JAVA_HOME
$env:MAVEN_HOME
$env:PATH -split ';'
~~~

If <code>where.exe mvn</code> says it cannot find Maven, inspect common locations:

~~~powershell
Get-ChildItem 'C:\Tools','C:\Program Files','C:\ProgramData\chocolatey' -Filter mvn.cmd -File -Recurse -ErrorAction SilentlyContinue | Select-Object -ExpandProperty FullName
~~~

Use the result to find Maven's **home folder** (the folder *above* <code>bin</code>). In **Manage Jenkins → Tools**, create/select a Maven installation and point it to that home folder, or add its <code>bin</code> directory to the Windows system PATH. **Restart the Jenkins service after changing the system PATH**:

~~~powershell
# Administrator PowerShell
Restart-Service Jenkins
~~~

Jenkins may run as a different Windows user from your PowerShell terminal. Verify commands **inside Jenkins Console Output**, not only locally. If Jenkins cannot connect to Docker Desktop, check <code>services.msc → Jenkins → Properties → Log On</code>; use a suitable account that can access Docker Desktop, with permission to use its engine, and restart Jenkins. Do not give an untrusted service access to the Docker engine.

**Mac — find paths:**

~~~bash
which -a java mvn git docker
command -v mvn
brew --prefix maven
/usr/libexec/java_home -V
echo "$PATH"
~~~

If the tested <code>/opt/maven/apache-maven-3.9.16/bin</code> path isn't present on another Mac, use the result of <code>which mvn</code> or your Maven installation's <code>bin</code> location in the Pipeline PATH.

### 7. GitHub webhook via ngrok

In Jenkins job **OLTRS-Pipeline → Configure → Build Triggers**, enable **GitHub hook trigger for GITScm polling**. Install the GitHub plugin if needed.

Start ngrok in a **separate terminal**, on either OS:

~~~bash
# Configure the token once; obtain it privately from your ngrok dashboard:
ngrok config add-authtoken YOUR_NGROK_TOKEN
ngrok http 8080
~~~

In **https://github.com/DevXOmar/OLTRS → Settings → Webhooks → Add webhook**:

| Setting | Value |
| --- | --- |
| Payload URL | <code>https://YOUR-NGROK-DOMAIN/github-webhook/</code> |
| Content type | **application/json** |
| Event | **Just the push event** |
| Active | Yes |
| SSL verification | Leave **enabled** for the HTTPS ngrok URL; no custom SSL certificate is needed |

Keep ngrok running for webhook tests; stop it after practice. The Jenkins server should require login. Run the Pipeline manually once so Jenkins registers the SCM.

**Test from your locally cloned OLTRS directory:**

~~~bash
git commit --allow-empty -m "Test Jenkins webhook"
git push origin main
~~~

Check **GitHub Webhooks → Recent Deliveries** and **Jenkins Build History**. If GitHub cannot reach Jenkins, verify the current ngrok URL, correct <code>/github-webhook/</code> suffix, webhook trigger checkbox, login/security settings, and branch <code>main</code>.

### 8. Optional email notification (not in the tested Pipeline)

The working script deliberately **does not send email**. For an email-specific question, install the **Email Extension** plugin, configure SMTP under **Manage Jenkins → System → Extended E-mail Notification**, and use a Jenkins-stored Gmail app password (if supported). With <code>smtp.gmail.com</code>, SSL port <code>465</code> is a common configuration. After SMTP works, optionally add <code>emailext</code> inside the Pipeline's <code>catch</code> block before <code>throw err</code>. Never place SMTP secrets in the Groovy script.

~~~groovy
// OPTIONAL: only if Email Extension + SMTP are configured
emailext(
    to: 'RECIPIENT_EMAIL_ADDRESS',
    subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
    body: "Check Jenkins Console Output: ${env.BUILD_URL}"
)
~~~

**Q2 scenarios:** no webhook = check delivery/trigger/ngrok; Maven failure = later stages skipped because the exception is rethrown; Docker Hub authentication failure = check <code>dockerhub-creds</code> and token permissions; deployment failure = check Console Output and <code>docker logs tomcat-local</code>; failure-only email = call <code>emailext</code> only from <code>catch</code>.

---

## Q1 (25M) — Docker Compose: OLTRS + Nagios

Q2 must already have published the image to **YOUR_DOCKERHUB_USERNAME/oltrs:latest**. Create a separate folder for Compose (**not inside the Maven build directory**).

### 1. Make the files

Windows PowerShell:

~~~powershell
mkdir set8-compose
cd set8-compose
notepad docker-compose.yml
# Write the YAML below, save, then:
notepad .env
~~~

Mac Terminal:

~~~bash
mkdir -p set8-compose
cd set8-compose
nano docker-compose.yml
# Write the YAML below, save, then:
nano .env
~~~

Contents of <code>.env</code> (only a public Docker Hub username; **not a token**):

~~~dotenv
DOCKERHUB_USER=YOUR_DOCKERHUB_USERNAME
~~~

Contents of <code>docker-compose.yml</code>:

~~~yaml
name: set8

services:
  web:
    image: ${DOCKERHUB_USER}/oltrs:latest
    container_name: set8-web
    ports:
      - "127.0.0.1:8088:8080"
    networks:
      - lab-net
    restart: unless-stopped

  nagios:
    image: jasonrivers/nagios:latest
    platform: linux/amd64
    container_name: set8-nagios
    ports:
      - "127.0.0.1:8888:80"
    networks:
      - lab-net
    depends_on:
      - web
    restart: unless-stopped

networks:
  lab-net:
    driver: bridge
~~~

The <code>platform: linux/amd64</code> line allows the Nagios image to run via emulation on Apple Silicon; it is also fine on Windows x64. Binding to <code>127.0.0.1</code> keeps these demo dashboards local.

### 2. Run and verify (same Docker commands on Windows/Mac)

~~~bash
docker compose config
docker compose pull
docker compose up -d
docker compose ps
docker compose logs
docker compose logs web
docker compose logs nagios
docker network ls
docker network inspect set8_lab-net
~~~

Open **http://localhost:8088** (OLTRS) and **http://localhost:8888/nagios/** (Nagios). The demo Nagios image commonly uses <code>nagiosadmin</code> / <code>nagios</code>; never publish it publicly with those credentials.

Check availability:

~~~bash
docker compose exec nagios getent hosts web
docker compose port web 8080
docker compose logs --tail=50 nagios
docker compose logs --tail=50 web
~~~

Check HTTP from the **host**:

~~~powershell
# Windows
curl.exe -I http://localhost:8088/
~~~

~~~bash
# Mac
curl -I http://localhost:8088/
~~~

**Important:** Running Nagios and OLTRS in one Compose network does **not** automatically configure an OLTRS Nagios health check. The <code>web</code> hostname is how Nagios would reach OLTRS internally at port <code>8080</code>; an explicit Nagios host/service check is required to show a real OLTRS UP/DOWN status.

### 3. Exact Q1 scenario commands

| Scenario | Command |
| --- | --- |
| Stop temporarily, keep containers/networks | <code>docker compose stop</code> |
| Restart stopped containers | <code>docker compose start</code> |
| Stop and remove containers/network | <code>docker compose down</code> |
| Add/change service and apply YAML | <code>docker compose up -d</code> |
| Web says UP but Nagios says DOWN | Check Nagios host/service address, internal port 8080, network and logs |

For Q1(e), run <code>docker compose ps</code>, <code>docker compose exec nagios getent hosts web</code>, <code>docker compose logs nagios</code>, <code>docker compose logs web</code>, and <code>docker network inspect set8_lab-net</code>. Use <code>curl.exe</code> (Windows) or <code>curl</code> (Mac) against port 8088.

---

## Q3 (25M) — Minikube: deploy, expose, scale to 4, diagnose, delete

Docker Desktop must be running. Commands below are the same on both systems. Replace the Docker Hub username with the one from Q2.

### 1. Start the cluster

~~~bash
minikube start --driver=docker
minikube status
kubectl cluster-info
kubectl get nodes
~~~

### 2. Deploy the Docker Hub image and verify

~~~bash
kubectl create deployment oltrs --image=YOUR_DOCKERHUB_USERNAME/oltrs:latest
kubectl get deployments
kubectl get pods
kubectl get pods -o wide
kubectl describe deployment oltrs
~~~

For Minikube to pull it without image credentials, the Docker Hub image must be publicly readable.

### 3. Expose with NodePort

~~~bash
kubectl expose deployment oltrs --type=NodePort --port=8080 --target-port=8080
kubectl get services
kubectl describe service oltrs
minikube service oltrs --url
~~~

Open the generated URL. If Minikube's Docker driver on Mac/Windows cannot reach that URL, open another terminal and run:

~~~bash
kubectl port-forward svc/oltrs 8081:8080
~~~

Then visit **http://localhost:8081**; leave that terminal open until done.

### 4. Scale to four Pods and verify

~~~bash
kubectl scale deployment oltrs --replicas=4
kubectl rollout status deployment/oltrs
kubectl get deployment oltrs
kubectl get pods -o wide
~~~

Expected: 4 desired and 4 available replicas, assuming the image runs successfully.

### 5. Troubleshooting and cleanup (SET-8 subparts)

~~~bash
# Four requested, fewer running / Pending / CrashLoopBackOff
kubectl get pods -o wide
kubectl describe pods
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl logs deployment/oltrs

# Service inaccessible / missing endpoints
kubectl get svc oltrs -o yaml
kubectl get pods --show-labels
kubectl get endpointslices -l kubernetes.io/service-name=oltrs
kubectl describe svc oltrs

# Remove lab Deployment and Service
kubectl delete deployment oltrs
kubectl delete service oltrs
kubectl get deployments
kubectl get services
kubectl get pods

# Optional: stop local cluster
minikube stop
~~~

If endpoints are empty, inspect Service selectors vs Pod labels and whether any Pods are Ready. Rollback for a separate update scenario (requires an older revision): <code>kubectl rollout undo deployment/oltrs</code>.

---

## Q4 (25M) — AWS Academy EC2 + OLTRS Docker image

### 1. Open the AWS Academy lab (college workflow)

1. Visit **https://awsacademy.instructure.com/** and log in.
2. **Courses → your course → Lab → Module → Launch AWS**.
3. Select **Start Lab**, then click **AWS** to open the temporary AWS console.
4. Go to **EC2 → Instances → Launch instances**.

Choose settings suitable for your lab's quota:

| Field | Setting |
| --- | --- |
| Name | OLTRS-Lab |
| AMI | Ubuntu Server, x86_64 |
| Instance type | Lab-allowed, suitably sized for Docker builds |
| Key pair | Create new RSA key pair, format **.pem**, save locally |
| Public IPv4 | Enabled |
| Storage | Enough space for Docker image layers/build (e.g. 16 GiB if permitted) |

**Inbound rules (critical):**

| Rule | Port | Source |
| --- | --- | --- |
| SSH | TCP 22 | My IP / authorised lab network |
| Application | Custom TCP **8080** | <code>0.0.0.0/0</code> for the short-lived browser demo |

Outbound internet access is needed for <code>apt</code>, Docker images and GitHub. Your AWS Academy session may impose service restrictions; use the instance options actually available in that lab.

### 2. Connect by SSH from your laptop

Use the instance's actual **Public IPv4** and the downloaded private key.

**Windows PowerShell:**

~~~powershell
cd "$HOME\Downloads"
ssh -i ".\OLTRS-Lab.pem" ubuntu@YOUR_PUBLIC_IP
~~~

If Windows OpenSSH complains that the key permissions are too open, from the key's directory:

~~~powershell
icacls ".\OLTRS-Lab.pem" /inheritance:r
icacls ".\OLTRS-Lab.pem" /grant:r "$($env:USERNAME):R"
ssh -i ".\OLTRS-Lab.pem" ubuntu@YOUR_PUBLIC_IP
~~~

**Mac Terminal:**

~~~bash
cd ~/Downloads
chmod 400 OLTRS-Lab.pem
ssh -i OLTRS-Lab.pem ubuntu@YOUR_PUBLIC_IP
~~~

All commands **below now run inside the Ubuntu EC2 SSH session**, not your laptop's local shell.

### 3. Install Docker and Git on Ubuntu

~~~bash
sudo apt update
sudo apt install -y docker.io git
sudo systemctl enable --now docker
sudo docker --version
sudo systemctl status docker --no-pager
~~~

### 4. Reliable re-run block (the fix used during practice)

This avoids cloning into the wrong directory or failing because <code>OLTRS</code> already exists:

~~~bash
# Go to home directory
cd ~

# Clone only if not already present
if [ ! -d OLTRS ]; then
    git clone https://github.com/DevXOmar/OLTRS.git
fi

# Enter the actual project folder
cd OLTRS

# Verify expected files
ls -la
cat Dockerfile

# Build image from the directory with Dockerfile
sudo docker build -t oltrs-ec2 .
~~~

If the existing folder is already a clone but needs newer commits, first check <code>git status</code>, then run <code>git pull --ff-only origin main</code> only when you have no conflicting local changes.

### 5. Run, check and open from laptop

~~~bash
sudo docker images
sudo docker run -d --name oltrs-ec2 --restart unless-stopped -p 8080:8080 oltrs-ec2
sudo docker ps
sudo docker logs oltrs-ec2
sudo docker port oltrs-ec2
curl -v http://127.0.0.1:8080/
~~~

**On your own laptop browser**, open **http://YOUR_PUBLIC_IP:8080**. If the container was already created, check <code>sudo docker ps -a</code> and run <code>sudo docker start oltrs-ec2</code> instead of duplicating the name.

**If building on a small EC2 instance runs out of memory:** use the prebuilt public image published in Q2 instead (only when the <code>oltrs-ec2</code> container doesn't already exist):

~~~bash
sudo docker pull YOUR_DOCKERHUB_USERNAME/oltrs:latest
sudo docker run -d --name oltrs-ec2 -p 8080:8080 YOUR_DOCKERHUB_USERNAME/oltrs:latest
~~~

### 6. All AWS scenario checks

| Scenario | Check |
| --- | --- |
| SSH fails | EC2 Running, Public IPv4, correct <code>.pem</code>, username <code>ubuntu</code>, SG TCP 22 from your IP |
| HTTP 8080 inaccessible | Inbound SG TCP 8080, Docker port map, local curl, firewall and listening sockets |
| No Public IPv4 | Public subnet/route, auto-assign public IPv4 or associated public address if allowed |
| Data must persist | Use retained EBS volume; stop/start is not the same as terminate |
| Public HTTP, private admin SSH | Open only app port 8080 publicly; limit TCP 22 to administrator IP |

Diagnostics **inside EC2**:

~~~bash
sudo docker ps -a
sudo docker logs oltrs-ec2
sudo docker port oltrs-ec2
curl -v http://localhost:8080/
sudo systemctl status docker --no-pager
sudo ufw status
sudo ss -lnt
~~~

If local <code>curl</code> works but the laptop browser cannot connect, focus on the AWS security group, public IPv4, routes and firewall.

### 7. Cleanup after screenshots

~~~bash
sudo docker stop oltrs-ec2
sudo docker rm oltrs-ec2
exit
~~~

In the AWS Academy console: **EC2 → Instances → OLTRS-Lab → Instance state → Terminate** (if permitted). End/stop the Academy lab according to its course controls, and check for any retained chargeable resources.

---

## Quick pass/fail evidence for the exam

| Question | Show examiner |
| --- | --- |
| **Q2** | Jenkins Stage View/Console Output green; Tomcat at **8085**; Docker Hub <code>oltrs:latest</code>; webhook delivery |
| **Q1** | Compose YAML; <code>docker compose ps</code>; OLTRS at **8088** and Nagios at **8888** |
| **Q3** | <code>kubectl get deployment,pods,svc</code>; **4 Pods**; working Minikube service or port-forward |
| **Q4** | EC2 Running + security group **8080**; Ubuntu SSH terminal; <code>docker ps</code>; browser on **PUBLIC_IP:8080** |

**Remember:** a GitHub repository stores source, a Docker image packages the app, Jenkins automates build/publish, Compose runs services together, Kubernetes manages replicas, Nagios monitors health, and EC2 hosts the deployed application remotely.
