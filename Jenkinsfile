pipeline {
    agent any

    tools {
        jdk 'jdk'
        nodejs 'node17'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-token',
                    url: 'https://github.com/prateekgajbar/Starbucks-DevSecOps-Platform.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=starbucks \
                    -Dsonar.projectKey=starbucks
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh '''
                        docker build -t starbucks .
                        docker tag starbucks pratikgajbar/starbucks:latest
                        docker push pratikgajbar/starbucks:latest
                        '''
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image pratikgajbar/starbucks:latest > trivyimage.txt'
            }
        }

        stage('App Deploy to Docker Container') {
            steps {
                sh '''
                docker rm -f starbucks || true
                docker run -d --name starbucks -p 3000:3000 pratikgajbar/starbucks:latest
                '''
            }
        }
    }

    post {
        always {
            script {
                def buildStatus = currentBuild.currentResult
                def buildUser = currentBuild
                    .getBuildCauses('hudson.model.Cause$UserIdCause')[0]
                    ?.userId ?: 'GitHub User'

                emailext(
                    subject: "Pipeline ${buildStatus}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                        <p>This is a Jenkins Starbucks CI/CD pipeline status.</p>
                        <p>Project: ${env.JOB_NAME}</p>
                        <p>Build Number: ${env.BUILD_NUMBER}</p>
                        <p>Build Status: ${buildStatus}</p>
                        <p>Started by: ${buildUser}</p>
                        <p>Build URL: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    """,
                    to: 'pratikgajbar@gmail.com',
                    from: 'pratikgajbar@gmail.com',
                    replyTo: 'pratikgajbar@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
                )
            }
        }
    }
}
