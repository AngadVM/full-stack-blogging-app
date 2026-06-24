pipeline {
    agent any
    tools {
        jdk "jdk"
        maven "maven"
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_NAME = "angadvm/ci-cd-pipeline-project"
        IMAGE_TAG = "latest"
    }
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/AngadVM/full-stack-blogging-app.git'
            }
        }
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        stage('Trivy FS Scan') {
            steps {
                sh "trivy fs . --format table -o fs.html"
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqubeServer') {
                    sh """
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=Blogging-app \
                    -Dsonar.projectKey=Blogging-app \
                    -Dsonar.java.binaries=target
                    """
                }
            }
        }
        stage('Build') {
            steps {
                sh "mvn package"
            }
        }
        stage('Publish Artifacts') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: 'jdk', maven: 'maven', mavenOpts: '-DskipTests', traceability: true) {
                    sh "mvn deploy"
                }
            }
        }
        stage('Docker Build') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t $IMAGE_NAME:$IMAGE_TAG ."
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --format table -o image.html $IMAGE_NAME:$IMAGE_TAG"
            }
        }
        stage('Docker Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push $IMAGE_NAME:$IMAGE_TAG"
                    }
                }
            }
        }
        stage('K8s Deploy') {
            steps {
               withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8s-token', namespace: 'webapps', serverUrl: 'https://2DF6EF6280BA8465041B60E3A3D73CC9.gr7.us-east-1.eks.amazonaws.com']]) {
                    sh "kubectl apply -f deployment-service.yml"
                    sleep 20
                }
            }
        }
        stage('Verify Deployment') {
            steps {
               withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8s-token', namespace: 'webapps', serverUrl: 'https://2DF6EF6280BA8465041B60E3A3D73CC9.gr7.us-east-1.eks.amazonaws.com']]) {
                    sh "kubectl get pods"
                    sh "kubectl get service"
                }
            }
        }
    }
}
post {
    always {
        script {
            def jobName = env.JOB_NAME
            def buildNumber = env.BUILD_NUMBER
            def pipelineStatus = currentBuild.result ?: 'SUCCESS'
            pipelineStatus = pipelineStatus.toUpperCase()

            def bannerColor = pipelineStatus == 'SUCCESS' ? 'green' : 'red'

            def body = """
            <body>
                <div style="border: 2px solid ${bannerColor}; padding: 15px;">
                    <h2 style="color: ${bannerColor};">
                         Jenkins Pipeline Status: ${pipelineStatus}
                    </h2>
                    <hr/>
                    <p><b>Job Name:</b> ${jobName}</p>
                    <p><b>Build Number:</b> ${buildNumber}</p>
                    <p><b>Status:</b> ${pipelineStatus}</p>
                    <p><b>Build URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                </div>
            </body>
            """

            emailext(
                subject: "${jobName} - Build #${buildNumber} - ${pipelineStatus}",
                body: body,
                to: 'angadvenugopal@gmail.com',
                mimeType: 'text/html',
                attachmentsPattern: 'fs.html, image.html'
            )
        }
    }
}