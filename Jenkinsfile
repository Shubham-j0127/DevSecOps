pipeline{
    agent any
    // environment {
    //     // SCANNER_HOME=tool 'sonar-scanner'
    //     TMDB_V3_API_KEY = credentials('tmdb-api-key')
    //     IMAGE_NAME = "sushmaagowdaa/netflix" // Name of the image created in Jenkins
    //     CONTAINER_NAME = "netflix" // Name of the container created in Jenkins
    // }
    stages {
        // stage('clean workspace'){
        //     steps{
        //         cleanWs()
        //     }
        // }
        stage('Checkout from Git'){
            steps{
                git 'https://github.com/Sushmaa123/DevSecOps-Project.git'
            }
        }
        // stage("Sonarqube Analysis "){
        //     steps{
        //         withSonarQubeEnv('sonar-server') {
        //             sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=DevSecOps-Project \
        //             -Dsonar.projectKey=DevSecOps-Project'''
        //         }
        //     }
        // }
       
        stage('OWASP FS SCAN') {
             steps {
             withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
            dependencyCheck additionalArguments: "--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey ${NVD_API_KEY}", odcInstallation: 'DP-Check'
             }
            dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
       }

        // stage('TRIVY FS SCAN') {
        //     steps {
        //         bat "trivy fs . > trivyfs.txt"     
        //     }
        // }
        // stage('Clean Up Docker Resources') {
        //     steps {
        //         script {
        //             // Remove the specific container
        //             bat '''
        //             if docker ps -a --format '{{.Names}}' | grep -q $CONTAINER_NAME; then
        //                 echo "Stopping and removing container: $CONTAINER_NAME"
        //                 docker stop $CONTAINER_NAME
        //                 docker rm $CONTAINER_NAME
        //             else
        //                 echo "Container $CONTAINER_NAME does not exist."
        //             fi
        //             '''

        //             // Remove the specific image
        //             bat '''
        //             if docker images -q $IMAGE_NAME; then
        //                 echo "Removing image: $IMAGE_NAME"
        //                 docker rmi -f $IMAGE_NAME
        //             else
        //                 echo "Image $IMAGE_NAME does not exist."
        //             fi
        //             '''
        //         }
        //     }
        // }
        // stage("Docker Build & Push"){
        //     steps{
        //         script{
        //            withDockerRegistry(credentialsId: 'docker-cred'){   
        //                bat 'docker build --build-arg TMDB_V3_API_KEY=$TMDB_V3_API_KEY -t $IMAGE_NAME .'
        //                bat 'docker push $IMAGE_NAME'
        //             }
        //         }
        //     }
        // }
        // stage("TRIVY"){
        //     steps{
        //         bat "trivy image $IMAGE_NAME > trivyimage.txt"
        //     }
        // }
        // stage('Deploy to container'){
        //     steps{
        //         bat 'docker run -itd --name $CONTAINER_NAME -p 8081:80 $IMAGE_NAME'
        //     }
        // }
    }
// post {
//      always {
//         emailext attachLog: true,
//             subject: "'${currentBuild.result}'",
//             body: "Project: ${env.JOB_NAME}<br/>" +
//                 "Build Number: ${env.BUILD_NUMBER}<br/>" +
//                 "URL: ${env.BUILD_URL}<br/>",
//             to: 'your-mail@gmail.com',                               
//             attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
//         }
//     }
}

