pipeline {
    agent any
    triggers {
        pollSCM('H/5 * * * *')
    }
    stages {
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'feature/pipeline'
            }
            steps {
                script {
                    app = docker.build("kallurusunil/train-schedule")
                    app.inside {
                        sh 'echo $(curl localhost:8080)'
                    }
                }
            }
        }
        stage('Clean Old Docker Images') {
            steps {
                script {
                    // Get the image ID of the latest image
                    def latestImage = sh(script: "docker images -q kallurusunil/train-schedule | head -n 1", returnStdout: true).trim()
                    
                    // List all images for the given repository
                    def allImages = sh(script: "docker images -q kallurusunil/train-schedule", returnStdout: true).trim().split('\n')
                    
                    // Filter out the latest image
                    def oldImages = allImages.findAll { it != latestImage }
                    
                    if (oldImages) {
                        echo "Removing old Docker images..."
                        oldImages.each { imageId ->
                            sh "docker rmi -f ${imageId}"
                        }
                    } else {
                        echo "No old Docker images to remove."
                    }
                }
            }
        }
        stage('List Docker Images') {
            steps {
                script {
                    sh 'docker images -q kallurusunil/train-schedule'
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'feature/pipeline'
            }
            steps {
                input 'Push to Docker hub?'
                milestone(1)
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'feature/pipeline'
            }
            steps {
                input 'Deploy to Production?'
                milestone(2)
                withCredentials([usernamePassword(credentialsId: 'webserver_login', usernameVariable: 'USERNAME', passwordVariable: 'USERPASS')]) {
                    script {
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker pull kallurusunil/train-schedule:${env.BUILD_NUMBER}\""
                        try {
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker stop train-schedule\""
                            sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker rm train-schedule\""
                        } catch (err) {
                            echo: 'caught error: $err'
                        }
                        sh "sshpass -p '$USERPASS' -v ssh -o StrictHostKeyChecking=no $USERNAME@$prod_ip \"docker run --restart always --name train-schedule -p 8080:8080 -d kallurusunil/train-schedule:${env.BUILD_NUMBER}\""
                    }
                }
            }
        }
    }
}
