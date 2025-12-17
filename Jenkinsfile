pipeline{
    agent any
    tools{
        jdk 'jdk'
        nodejs 'node17'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {

        stage('Clean Workspace'){
            steps{
                cleanWs()
            }
        }

        stage('Checkout from Git'){
            steps{
                git branch: 'main',
                    credentialsId: 'github-token',
                    url: 'https://github.com/akhtaralikhan786/starbucks-demo.git'
            }
        }

        stage("Sonarqube Analysis"){
            steps{
                withSonarQubeEnv('SonarQube') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=starbucks \
                    -Dsonar.projectKey=starbucks
                    '''
                }
            }
        }

        stage("Quality Gate"){
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        stage('Install Dependencies'){
            steps {
                sh "npm install"
            }
        }

        stage('TRIVY FS SCAN'){
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage("Docker Build & Push"){
            steps{
                script{
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                        sh "docker build -t starbucks ."
                        sh "docker tag starbucks akhtaralikhan171184/starbucks:latest"
                        sh "docker push akhtaralikhan171184/starbucks:latest"
                    }
                }
            }
        }

        stage("TRIVY IMAGE SCAN"){
            steps{
                sh "trivy image akhtaralikhan171184/starbucks:latest > trivyimage.txt"
            }
        }

        stage('Remove Old Starbucks Container'){
            steps {
                sh '''
                if docker ps -a --format '{{.Names}}' | grep -w starbucks > /dev/null; then
                    echo "Stopping and removing old container: starbucks"
                    docker stop starbucks || true
                    docker rm starbucks || true
                else
                    echo "No existing container named starbucks found"
                fi
                '''
            }
        }

        stage('App Deploy to Docker container'){
            steps{
                sh '''
                docker run -d \
                --name starbucks \
                -p 3001:3000 \
                akhtaralikhan171184/starbucks:latest
                '''
            }
        }
    }

    post {
        always {
            script {
                def buildStatus = currentBuild.currentResult
                def buildUser = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')[0]?.userId ?: 'Github User'

                emailext (
                    subject: "Pipeline ${buildStatus}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                        <p>This is a Jenkins Starbucks CI/CD pipeline status.</p>
                        <p>Project: ${env.JOB_NAME}</p>
                        <p>Build Number: ${env.BUILD_NUMBER}</p>
                        <p>Build Status: ${buildStatus}</p>
                        <p>Started by: ${buildUser}</p>
                        <p>Build URL: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    """,
                    to: 'akhtaralikhan786@gmail.com',
                    from: 'akhtaralikhan786@gmail.com',
                    replyTo: 'akhtaralikhan786@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
                )
            }
        }
    }
}
