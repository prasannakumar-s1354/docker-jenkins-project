pipeline {
    agent any

    stages {

        stage('Docker Check') {
            steps {
                bat 'docker --version'
                bat 'docker info'
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build -t docker-jenkins-app:latest .'
            }
        }

        stage('Docker Hub Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-pat-test',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    powershell '''
                        $tempFile = "$env:TEMP\\docker-pat.txt"

                        [System.IO.File]::WriteAllText(
                            $tempFile,
                            $env:DOCKER_PASSWORD,
                            [System.Text.UTF8Encoding]::new($false)
                        )

                        docker logout

                        cmd /c "docker login -u $env:DOCKER_USERNAME --password-stdin < `"$tempFile`""

                        $loginResult = $LASTEXITCODE

                        Remove-Item $tempFile -Force

                        if ($loginResult -ne 0) {
                            exit $loginResult
                        }
                    '''
                }
            }
        }

        stage('Tag and Push Docker Image') {
            steps {
                bat 'docker tag docker-jenkins-app:latest spk1354/docker-jenkins-app:latest'
                bat 'docker push spk1354/docker-jenkins-app:latest'
            }
        }

        stage('Run Built Image') {
            steps {
                bat 'docker rm -f docker-jenkins-app-jenkins 2>NUL || echo No old test container found'
                bat 'docker run -d -p 8084:80 --name docker-jenkins-app-jenkins docker-jenkins-app:latest'
            }
        }

        stage('Docker Container Check') {
            steps {
                bat 'docker ps'
                bat 'docker inspect docker-jenkins-app-jenkins'
            }
        }

        stage('Volume Persistence Test') {
            steps {

                bat 'docker exec docker-jenkins-app cat /usr/share/nginx/html/volume-test.txt'

                bat 'docker stop docker-jenkins-app'

                bat 'docker rm docker-jenkins-app'

                bat 'docker run -d -p 8081:80 --name docker-jenkins-app -v jenkins-docker-volume:/usr/share/nginx/html nginx'

                bat 'docker exec docker-jenkins-app cat /usr/share/nginx/html/volume-test.txt'
            }
        }
    }
}