# Deploy Netflix Clone on Cloud using Jenkins - DevSecOps Project

## Phase 1: Initial Setup and Deployment

### Step 1: Launch EC2 (Ubuntu 22.04)
- Provision an EC2 instance with:
  - Instance type: `t2.medium`
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
  docker rm -f netflix
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
sudo docker run -itd --name sonarqube -p 9000:9000 sonarqube:lts-community
```
- Access SonarQube at: `http://<public-ip>:9000` (Default credentials: admin/admin)
- 
## Phase 3: CI/CD Setup to Run Netflix Using Jenkins

### Step 1: Install Jenkins for Automation
- Install Java:
  ```bash
  sudo apt update
  sudo apt install fontconfig openjdk-21-jre -y
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
  sudo cat /var/jenkins_home/secrets/initialAdminPassword
  ```
- Paste the password in Jenkins setup, install suggested plugins, and create a user.
- Give docker permission to jenkins user:
  ``` sudo usermod -aG docker jenkins ```

### Step 2: Install Necessary Plugins in Jenkins
- Go to `Manage Jenkins` → `Plugins` → `Available Plugins`.
- Install the following plugins:
  1. SonarQube Scanner
  2. OWASP Dependency Check
  3. Docker, Docker Pipeline, Docker Build-Step, CloudBees Docker Build & Publish
  4. Pipeline Stage View

### Step 3: Configure Java, Node.js, sonar-scanner,owasp dependency check and docker in Global Tool Configuration

 Global Tool Configuration is used to configure different tools that we install using Plugins

- Go to `Manage Jenkins` → `Tools` → Install:
  - Nodejs16
  - Sonar-scanner
  - DP-Check
  - docker
After adding all the above names in the respective section, select install automatically and add your desired version and installation method
- Click `Apply` and `Save`.

### step 4: Configure Sonarqube

The Configure System option is used in Jenkins to configure different server

- Go to sonarqube server and create a token
  
  - go to `administrator` -> `security` -> `users` -> `token`

- Go to system configure in jenkins
  
  **Sonarqube**

   - select environment variables
   - Add name [sonar-server] and add credentials


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
       
        stage('OWASP FS SCAN') {
             steps {
             withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
            dependencyCheck additionalArguments: "--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey ${NVD_API_KEY}", odcInstallation: 'DP-Check'
             }
            dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
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
        stage('Trivy Scan') {
            steps {
                sh '''
                    docker run --rm \
                      -v /var/run/docker.sock:/var/run/docker.sock \
                      aquasec/trivy:latest \
                      image myapp:${BUILD_NUMBER} > trivy-image.txt
                '''
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
            attachmentsPattern: 'trivyimage.txt'
        }
    }
}

```
