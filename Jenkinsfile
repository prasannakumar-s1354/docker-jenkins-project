pipeline {
    agent any

    environment {
        ECR_REGISTRY = '018913575233.dkr.ecr.ap-southeast-2.amazonaws.com'
        ECR_REPOSITORY = 'docker-jenkins-app'
        ECR_IMAGE = '018913575233.dkr.ecr.ap-southeast-2.amazonaws.com/docker-jenkins-app:latest'
    }

    stages {

        stage('Docker Check') {
            steps {
                bat 'docker --version'
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'cd /d C:\\docker-jenkins-project && docker build -t docker-jenkins-app:latest .'
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
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-pat-test',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {

                    bat 'docker tag docker-jenkins-app:latest %DOCKER_USERNAME%/docker-jenkins-app:latest'

                    bat 'docker push %DOCKER_USERNAME%/docker-jenkins-app:latest'
                }
            }
        }

        stage('ECR Login and Push') {
            steps {

                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-ecr-jenkins'
                ]]) {

                    powershell '''
                        $ErrorActionPreference = "Stop"

                        $config = Join-Path $env:WORKSPACE ".docker-ecr"

                        try {

                            # Remove old temporary Docker configuration
                            if (Test-Path $config) {
                                Remove-Item $config -Recurse -Force
                            }

                            # Create clean Docker configuration folder
                            New-Item -ItemType Directory -Path $config | Out-Null

                            Write-Host "Getting ECR login password..."

                            $password = aws ecr get-login-password --region ap-southeast-2

                            if ($LASTEXITCODE -ne 0) {
                                throw "AWS ECR login password generation failed."
                            }

                            Write-Host "Logging into Amazon ECR..."

                            $password | docker --config $config login `
                                --username AWS `
                                --password-stdin `
                                $env:ECR_REGISTRY

                            if ($LASTEXITCODE -ne 0) {
                                throw "Docker login to ECR failed."
                            }

                            Write-Host "Tagging image for ECR..."

                            docker --config $config tag `
                                docker-jenkins-app:latest `
                                $env:ECR_IMAGE

                            if ($LASTEXITCODE -ne 0) {
                                throw "ECR image tagging failed."
                            }

                            Write-Host "Pushing image to ECR..."

                            docker --config $config push $env:ECR_IMAGE

                            if ($LASTEXITCODE -ne 0) {
                                throw "ECR image push failed."
                            }

                            Write-Host "ECR image pushed successfully."

                        }
                        finally {

                            # Remove temporary Docker credentials
                            if (Test-Path $config) {
                                Remove-Item $config -Recurse -Force
                            }
                        }
                    '''
                }
            }
        }

        stage('Pull Image from Docker Hub') {
            steps {
                bat 'docker pull spk1354/docker-jenkins-app:latest'
            }
        }

        stage('Run Docker Hub Image') {
            steps {
                bat 'docker rm -f docker-jenkins-app-jenkins 2>NUL || echo No old test container found'

                bat 'docker run -d -p 8084:80 --name docker-jenkins-app-jenkins spk1354/docker-jenkins-app:latest'
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
