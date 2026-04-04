# Deploy Netflix Clone on Cloud using Jenkins - DevSecOps Project

## Phase 1: Initial Setup and Deployment

### Step 1: Launch EC2 (Ubuntu 22.04)
- Provision an EC2 instance with:
  - Instance type: `t2.large`
  - Volume: `15GB`
- Connect to the instance using SSH.

### Step 2: Clone the Code
- Update all packages and clone the repository:
  ```bash
  sudo apt-get update
  git clone https://github.com/Sushmaa123/DevSecOps-Project.git
  ```

### Step 3: Install Docker and Run the App Using a Container
- Install Docker:
  ```bash
  sudo apt-get update
  sudo apt-get install docker.io -y
  sudo usermod -aG docker $USER  # Replace with your system's username, e.g., 'ubuntu'
  newgrp docker
  sudo chmod 777 /var/run/docker.sock
  ```

- Build and run the application using Docker:
  ```bash
  docker build -t netflix .
  docker run -d --name netflix -p 8081:80 netflix:latest
  ```

- To stop and remove the container:
  ```bash
  docker stop <containerid>
  docker rmi -f netflix
  ```

**Note:** You need an API key for the application to work.

### Step 4: Get the API Key
- Open a web browser and navigate to [TMDB (The Movie Database)](https://www.themoviedb.org/).
- Log in or create an account.
- Navigate to `Settings` > `API`.
- Create a new API key and accept the terms.
- Use the API key when building the Docker image:
  ```bash
  docker build --build-arg TMDB_V3_API_KEY=<your-api-key> -t netflix .
  ```

- Access the application at:
  ```
  http://<ip-address>:8081
  ```

## Phase 2: Install SonarQube and Trivy
### Install SonarQube
```bash
sudo docker run -itd --name sonarqube -p 9000:9000 sonarqube
```
- Access SonarQube at: `http://<public-ip>:9000` (Default credentials: admin/admin)

### Install Trivy
```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```
- Scan an image using Trivy:
  ```bash
  trivy image <imageid>
  ```

## Phase 3: CI/CD Setup to Run Netflix Using Jenkins

### Step 1: Install Jenkins for Automation
- Install Java:
  ```bash
sudo apt update
sudo apt install fontconfig openjdk-21-jre
java -version
  ```
- Install Jenkins:
  ```bash
  sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
  echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
  sudo apt update
  sudo apt install jenkins -y
  ```
- Access Jenkins in a web browser:
  ```
  http://<public-ip>:8080
  ```
- Retrieve the administrator password:
  ```bash
  cat /var/jenkins_home/secrets/initialAdminPassword
  ```
- Paste the password in Jenkins setup, install suggested plugins, and create a user.

### Step 2: Install Necessary Plugins in Jenkins
- Go to `Manage Jenkins` → `Plugins` → `Available Plugins`.
- Install the following plugins:
  1. SonarQube Scanner
  2. OWASP Dependency Check
  3. NodeJS Plugin
  4. Docker, Docker Pipeline, Docker Build-Step, CloudBees Docker Build & Publish
  5. Slack Notification
  6. Pipeline Stage View

### Step 3: Configure Java, Node.js, sonar-scanner,owasp dependency check and docker in Global Tool Configuration

 Global Tool Configuration is used to configure different tools that we install using Plugins

- Go to `Manage Jenkins` → `Tools` → Install:
  - Nodejs16
  - Sonar-scanner
  - DP-Check
  - docker
After adding all the above names in the respective section, select install automatically and add your desired version and installation method
- Click `Apply` and `Save`.

### step 4: Configure Sonarqube and Slack in System Configuration

The Configure System option is used in Jenkins to configure different server

- Go to sonarqube server and create a token
  
  - go to `administrator` -> `security` -> `users` -> `token`

- Go to system configure in jenkins
  
  **Sonarqube**

   - select environment variables
   - Add name [sonar-server] and add credentials

  **Slack**

    - Create a workspace and add channel
    - Go to slack app and add jenkins CI to slack
    - Get the subdomain and credentialsID
    - add subdomain and credentials in jenkins
    - Click on apply and save

### step 5: Add all the required credentials in security credentials section

   - Go to `credentials` -> `global` -> `add credentials`
   - add credentials for below list:
      - nvd-api-key
      - tmdb-api-key
      - docker-cred

### step 6: Create a Pipeline Job and configure it

   - Go to dashboard of jenkins
   - click on new item and give name for the job then select pipeline job
   - Create jenkins webhook to automatically trigger the changes

      - in the build triggers select githubhook trigger for scm
      - then go to your github repository, open settings and select webhook
      - add payload url then select application/json in content type and save it

   - now write a pipeline code

   ```
   pipeline{
    agent any
    tools{
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
        NVD_API_KEY = credentials('nvd-api-key')
        TMDB_V3_API_KEY = credentials('tmdb-api-key')
        IMAGE_NAME = "sushmaagowdaa/netflix" // Name of the image created in Jenkins
        CONTAINER_NAME = "netflix" // Name of the container created in Jenkins
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git 'https://github.com/Sushmaa123/DevSecOps-Project.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=DevSecOps-Project \
                    -Dsonar.projectKey=DevSecOps-Project'''
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
             withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
            dependencyCheck additionalArguments: "--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey ${NVD_API_KEY}", odcInstallation: 'DP-Check'
             }
            dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
       }

        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"     
            }
        }
        stage('Clean Up Docker Resources') {
            steps {
                script {
                    // Remove the specific container
                    sh '''
                    if docker ps -a --format '{{.Names}}' | grep -q $CONTAINER_NAME; then
                        echo "Stopping and removing container: $CONTAINER_NAME"
                        docker stop $CONTAINER_NAME
                        docker rm $CONTAINER_NAME
                    else
                        echo "Container $CONTAINER_NAME does not exist."
                    fi
                    '''

                    // Remove the specific image
                    sh '''
                    if docker images -q $IMAGE_NAME; then
                        echo "Removing image: $IMAGE_NAME"
                        docker rmi -f $IMAGE_NAME
                    else
                        echo "Image $IMAGE_NAME does not exist."
                    fi
                    '''
                }
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker-cred'){   
                       sh 'docker build --build-arg TMDB_V3_API_KEY=$TMDB_V3_API_KEY -t $IMAGE_NAME .'
                       sh 'docker push $IMAGE_NAME'
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image $IMAGE_NAME > trivyimage.txt"
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -itd --name $CONTAINER_NAME -p 8081:80 $IMAGE_NAME'
            }
        }
    }
post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'your-mail@gmail.com',                               
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}

```
    
# Phase 4: Monitoring

## Install Prometheus and Grafana

Set up Prometheus and Grafana to monitor your application.

---
**Phase 4: Monitoring**

1. **Install Prometheus and Grafana:**

   Set up Prometheus and Grafana to monitor your application.

   **Installing Prometheus:**

   First, create a dedicated Linux user for Prometheus and download Prometheus:

   ```bash
   sudo useradd --system --no-create-home --shell /bin/false prometheus
   wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
   ```

   Extract Prometheus files, move them, and create directories:

   ```bash
   tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
   cd prometheus-2.47.1.linux-amd64/
   sudo mkdir -p /data /etc/prometheus
   sudo mv prometheus promtool /usr/local/bin/
   sudo mv consoles/ console_libraries/ /etc/prometheus/
   sudo mv prometheus.yml /etc/prometheus/prometheus.yml
   ```

   Set ownership for directories:

   ```bash
   sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
   ```

   Create a systemd unit configuration file for Prometheus:

   ```bash
   sudo nano /etc/systemd/system/prometheus.service
   ```

   Add the following content to the `prometheus.service` file:

   ```plaintext
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
   ```

   Here's a brief explanation of the key parts in this `prometheus.service` file:

   - `User` and `Group` specify the Linux user and group under which Prometheus will run.

   - `ExecStart` is where you specify the Prometheus binary path, the location of the configuration file (`prometheus.yml`), the storage directory, and other settings.

   - `web.listen-address` configures Prometheus to listen on all network interfaces on port 9090.

   - `web.enable-lifecycle` allows for management of Prometheus through API calls.

   Enable and start Prometheus:

   ```bash
   sudo systemctl enable prometheus
   sudo systemctl start prometheus
   ```

   Verify Prometheus's status:

   ```bash
   sudo systemctl status prometheus
   ```

   You can access Prometheus in a web browser using your server's IP and port 9090:

   `http://<your-server-ip>:9090`

   **Installing Node Exporter:**

   Create a system user for Node Exporter and download Node Exporter:

   ```bash
   sudo useradd --system --no-create-home --shell /bin/false node_exporter
   wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
   ```

   Extract Node Exporter files, move the binary, and clean up:

   ```bash
   tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
   sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
   rm -rf node_exporter*
   ```

   Create a systemd unit configuration file for Node Exporter:

   ```bash
   sudo nano /etc/systemd/system/node_exporter.service
   ```

   Add the following content to the `node_exporter.service` file:

   ```plaintext
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
   ```

   Replace `--collector.logind` with any additional flags as needed.

   Enable and start Node Exporter:

   ```bash
   sudo systemctl enable node_exporter
   sudo systemctl start node_exporter
   ```

   Verify the Node Exporter's status:

   ```bash
   sudo systemctl status node_exporter
   ```

   You can access Node Exporter metrics in Prometheus.

2. **Configure Prometheus Plugin Integration:**

   Integrate Jenkins with Prometheus to monitor the CI/CD pipeline.

   **Prometheus Configuration:**

   To configure Prometheus to scrape metrics from Node Exporter and Jenkins, you need to modify the `prometheus.yml` file. Here is an example `prometheus.yml` configuration for your setup:

   ```yaml
   global:
     scrape_interval: 15s

   scrape_configs:
     - job_name: 'node_exporter'
       static_configs:
         - targets: ["localhost:9100"]

     - job_name: 'jenkins'
       metrics_path: '/prometheus'
       static_configs:
         - targets: ["<your-jenkins-ip>:<your-jenkins-port>"]
   ```

   Make sure to replace `<your-jenkins-ip>` and `<your-jenkins-port>` with the appropriate values for your Jenkins setup.

   Check the validity of the configuration file:

   ```bash
   promtool check config /etc/prometheus/prometheus.yml
   ```

   Reload the Prometheus configuration without restarting:

   ```bash
   curl -X POST http://localhost:9090/-/reload
   ```

   You can access Prometheus targets at:

   `http://<your-prometheus-ip>:9090/targets`


####Grafana

**Install Grafana on Ubuntu 22.04 and Set it up to Work with Prometheus**

**Step 1: Install Dependencies:**

First, ensure that all necessary dependencies are installed:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https software-properties-common
```

**Step 2: Add the GPG Key:**

Add the GPG key for Grafana:

```bash
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
```

**Step 3: Add Grafana Repository:**

Add the repository for Grafana stable releases:

```bash
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
```

**Step 4: Update and Install Grafana:**

Update the package list and install Grafana:

```bash
sudo apt-get update
sudo apt-get -y install grafana
```

**Step 5: Enable and Start Grafana Service:**

To automatically start Grafana after a reboot, enable the service:

```bash
sudo systemctl enable grafana-server
```

Then, start Grafana:

```bash
sudo systemctl start grafana-server
```

**Step 6: Check Grafana Status:**

Verify the status of the Grafana service to ensure it's running correctly:

```bash
sudo systemctl status grafana-server
```

**Step 7: Access Grafana Web Interface:**

Open a web browser and navigate to Grafana using your server's IP address. The default port for Grafana is 3000. For example:

`http://<your-server-ip>:3000`

You'll be prompted to log in to Grafana. The default username is "admin," and the default password is also "admin."

**Step 8: Change the Default Password:**

When you log in for the first time, Grafana will prompt you to change the default password for security reasons. Follow the prompts to set a new password.

**Step 9: Add Prometheus Data Source:**

To visualize metrics, you need to add a data source. Follow these steps:

- Click on the gear icon (⚙️) in the left sidebar to open the "Configuration" menu.

- Select "Data Sources."

- Click on the "Add data source" button.

- Choose "Prometheus" as the data source type.

- In the "HTTP" section:
  - Set the "URL" to `http://localhost:9090` (assuming Prometheus is running on the same server).
  - Click the "Save & Test" button to ensure the data source is working.

**Step 10: Import a Dashboard:**

To make it easier to view metrics, you can import a pre-configured dashboard. Follow these steps:

- Click on the "+" (plus) icon in the left sidebar to open the "Create" menu.

- Select "Dashboard."

- Click on the "Import" dashboard option.

- Enter the dashboard code you want to import (e.g., code 1860).

- Click the "Load" button.

- Select the data source you added (Prometheus) from the dropdown.

- Click on the "Import" button.

You should now have a Grafana dashboard set up to visualize metrics from Prometheus.

Grafana is a powerful tool for creating visualizations and dashboards, and you can further customize it to suit your specific monitoring needs.

That's it! You've successfully installed and set up Grafana to work with Prometheus for monitoring and visualization.

2. **Configure Prometheus Plugin Integration:**
    - Integrate Jenkins with Prometheus to monitor the CI/CD pipeline.
