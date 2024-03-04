pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
        maven 'maven3'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }                                                     //  Repository URL need to be updated here
        stage('Checkout from Git') {                          //                    |
            steps {                                           //                    |
                git branch: 'main', url: 'https://github.com/yash509/DevSecOps-Project----Animal-Farm-Deployment.git'
            }
        }
        stage ('maven compile') {
            steps {
                sh 'mvn clean compile'
            }
        }
        stage ('maven Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage("Sonarqube Analysis") {                         // Project Name need to be change here
            steps {                                           //                 |
                withSonarQubeEnv('sonar-server') {            //                 |
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=animal \
                    -Dsonar.projectKey=animal''' // -- Project Key need to be change here
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
        stage ('Build war file'){
            steps{
                sh 'mvn clean install -DskipTests=true'
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
                       sh "docker build -t animal-farm ." // -- Docker image Name to be update here
                       sh "docker tag animal-farm yash5090/animal-farm:latest " // -- Docker image Name to be update here too
                       sh "docker push yash5090/animal-farm:latest " // -- Docker image Name to be update here too
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image yash5090/animal-farm:latest > trivyimage.txt"  // -- Docker image Name to be update here 
            }
        }
        stage('Manual Approval') {
          timeout(time: 10, unit: 'MINUTES') {
            mail to: 'postbox.aj99@gmail.com', // -- E-mail to be updated here 
              subject: "${currentBuild.result} CI: ${env.JOB_NAME}",
              body: "Project: ${env.JOB_NAME}\nBuild Number: ${env.BUILD_NUMBER}\nGo to ${env.BUILD_URL} and approve deployment"
            input message: "Deploy ${params.project_name}?", 
              id: "DeployGate", 
              submitter: "approver", 
              parameters: [choice(name: 'action', choices: ['Deploy'], description: 'Approve deployment')]
          }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name farm -p 5000:5000 yash5090/animal-farm:latest' // -- Docker image Name to be update here
            }
        }
        stage('K8s'){
          steps{
            script{
              withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                sh 'kubectl apply -f deployment.yaml'
                sh 'kubectl apply -f service.yaml' // -- if needed
              }
            }
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
            to: 'postbox.aj99@gmail.com', // -- E-mail to be updated here 
            attachmentsPattern: 'trivy.txt'
        }
    }
}
