pipeline {
    agent any
    tools {
        jdk 'java-17'
        maven 'maven3'
    }
    environment {
        SCANNER_HOME= tool 'sonar-scanner'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'gitcreds', url: 'https://github.com/SaranKartheek/Board_Game.git'
            }
        }
        stage('Compile') {
            steps {
                script{
                 sh "mvn compile"   
                }
            }
        }
        stage('Test') {
            steps {
                script {
                    sh "mvn test"
                }
            }
        }
        stage('File System Scan') {
            steps {
                script {
                    sh "trivy fs --format table -o trivy-fs.html ."
                }
            }
        }
        stage('Sonarqube Analysis') {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'sonartoken') {
                        sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName='Boardgame' -Dsonar.projectKey='Boardgame' -Dsonar.java.libraries=."
                    }
                }
            }
        }
        stage('Quality Gates') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonartoken'
                }
            }
        }
        stage('Build') {
            steps {
                script {
                    sh "mvn package"
                }
            }
        }
        stage('Push to Nexus') {
            steps {
                script {
                    withMaven(globalMavenSettingsConfig: 'maven_settings_file', jdk: 'java-17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                        sh "mvn deploy"
                    }
                }
            }
        }
        stage('Build and Tag Docker Image') {
            steps {
                script {
                   withDockerRegistry(credentialsId: 'dockercreds', toolName: 'docker') {
                       sh "docker build -t saranvemireddy/boardgame:latest ."
                   }
                }
            }
        }
        stage('Docker Image Scan') {
            steps {
                script {
                   sh "trivy image --format table -o trivy-image.html saranvemireddy/boardgame:latest"
                }
            }
        }
        stage('Push to Docker Hub') {
            steps {
                script {
                   withDockerRegistry(credentialsId: 'dockercreds', toolName: 'docker') {
                       sh "docker push saranvemireddy/boardgame:latest"
                   }
                }
            }
        }
        stage('Deploy to K8') {
            steps {
                script {
                   withKubeConfig(caCertificate: '', clusterName: 'boardgame.us-east-1.eksctl.io', contextName: '', credentialsId: 'k8creds', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://16F244B7DC32B759A338058C1D0D50AE.gr7.us-east-1.eks.amazonaws.com') {
                       sh "kubectl apply -f deployment-service.yaml"
                   }
                }
            }
        }
        stage('Verify the Deployment') {
            steps {
                script {
                   withKubeConfig(caCertificate: '', clusterName: 'boardgame.us-east-1.eksctl.io', contextName: '', credentialsId: 'k8creds', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://16F244B7DC32B759A338058C1D0D50AE.gr7.us-east-1.eks.amazonaws.com') {
                       sh "kubectl get pods -n webapps"
                       sh "kubectl get svc -n webapps"
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
            def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
            def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

            def body = """
                <html>
                <body>
                <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                <h2>${jobName} - Build ${buildNumber}</h2>
                <div style="background-color: ${bannerColor}; padding: 10px;">
                <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                </div>
                <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                </div>
                </body>
                </html>
            """

            emailext (
                subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                body: body,
                to: 'sarankartheek1525@gmail.com',
                from: 'jenkins@example.com',
                replyTo: 'jenkins@example.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy-image.html'
            )
            }
        }
    }
}