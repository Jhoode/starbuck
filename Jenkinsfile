pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        // >>> CHANGE THESE TWO <<<
        GIT_REPO   = 'https://github.com/Jhoode/starbuck.git'
        IMAGE      = '<your-dockerhub-user>/starbucks:latest'
    }
    stages {
        stage('Clean workspace') { steps { cleanWs() } }

        stage('Git checkout') {
            steps { git branch: 'main', url: "${GIT_REPO}" }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=starbucks -Dsonar.projectKey=starbucks'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script { waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' }
            }
        }

        stage('Install NPM Dependencies') { steps { sh 'npm ci' } }

        stage('OWASP FS Scan') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy File Scan') { steps { sh 'trivy fs . > trivy.txt' } }

        stage('Build Docker Image') { steps { sh 'docker build -t starbucks .' } }

        stage('Tag & Push to DockerHub') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker') {
                        sh "docker tag starbucks ${IMAGE}"
                        sh "docker push ${IMAGE}"
                    }
                }
            }
        }

        stage('Docker Scout Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker-scout quickview ${IMAGE}"
                        sh "docker-scout cves ${IMAGE}"
                        sh "docker-scout recommendations ${IMAGE}"
                    }
                }
            }
        }

        stage('Deploy to Container') {
            steps {
                // remove the previous run so re-deploys don't fail on the name clash
                sh 'docker rm -f starbucks || true'
                sh "docker run -d --name starbucks -p 3000:3000 ${IMAGE}"
            }
        }
    }
    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: "Project: ${env.JOB_NAME}<br/>Build: ${env.BUILD_NUMBER}<br/>URL: ${env.BUILD_URL}",
                to: 'ojugo.jude@gmail.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy.txt'
        }
    }
}
