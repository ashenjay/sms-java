pipeline {
    agent any
    
    tools {
        jdk 'jdk'
        maven 'maven3'
    }
    
    environment {
        SCANNER_HOME = tool 'SonarScanner'
    }
    
    stages {
        stage('Git Checkout') {
            steps {
                git 'https://github.com/ashenjay/sms-java.git'
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Test') {
            steps {
                sh "mvn test"
            }
        }
        
        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o fs.html ."
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('Sonar-Scanner') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=sms-java -Dsonar.projectKey=sms-java \
                          -Dsonar.java.binaries=target'''
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script {
                    def qualityGate = waitForQualityGate()
                    if (qualityGate.status != 'OK') {
                        error "Pipeline aborted due to quality gate failure: ${qualityGate.status}"
                    }
                }
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn package"
            }
        }
        
        stage('Publish Artifacts to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: 'jdk', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy"
                }
            }
        }
        
        stage('Docker Build') {
            steps {
                script {
                    docker.withRegistry('', 'docker-cred') {
                        sh "docker build -t ashenjay/sms-java:latest ."
                    }
                }
            }
        }
        
        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --format table -o image.html ashenjay/sms-java:latest"
            }
        }
        
        stage('Docker Push Image') {
            steps {
                script {
                    docker.withRegistry('', 'docker-cred') {
                        sh "docker push ashenjay/sms-java:latest"
                    }
                }
            }
        }
        
        stage('K8-Deploy') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'terraform-cluster', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://CD3119C1398A73573D7EA2B43D8CD876.gr7.us-east-1.eks.amazonaws.com') {
                    sh "kubectl apply -f deployment-service.yaml"
                    sleep 20
                }
            }
        }
        
        stage('K8-Verify the Deployment') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'terraform-cluster', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://CD3119C1398A73573D7EA2B43D8CD876.gr7.us-east-1.eks.amazonaws.com') {
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
                }
            }
        }
    }
    
    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'SUCCESS' // Set default to SUCCESS
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

                emailext(
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'ashenjanath@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html'
                )
            }
        }
    }
}
