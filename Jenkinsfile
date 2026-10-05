pipeline {

    agent any

    stages {

        stage('Docker Check') {
            steps {
                powershell '''
                    Write-Host "========================================"
                    Write-Host "Checking Docker..."
                    Write-Host "========================================"

                    docker --version
                    docker info

                    if ($LASTEXITCODE -ne 0) {
                        throw "Docker check failed."
                    }
                '''
            }
        }


        stage('Build Docker Image') {
            steps {
                powershell '''
                    Write-Host "========================================"
                    Write-Host "Building Docker Image..."
                    Write-Host "========================================"

                    docker build -t docker-jenkins-app:latest .

                    if ($LASTEXITCODE -ne 0) {
                        throw "Docker image build failed."
                    }

                    docker images docker-jenkins-app
                '''
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
                        $ErrorActionPreference = "Stop"

                        Write-Host "========================================"
                        Write-Host "Logging into Docker Hub..."
                        Write-Host "========================================"

                        $tempFile = "$env:TEMP\\docker-pat.txt"

                        try {

                            [System.IO.File]::WriteAllText(
                                $tempFile,
                                $env:DOCKER_PASSWORD,
                                [System.Text.UTF8Encoding]::new($false)
                            )

                            docker logout 2>$null

                            cmd /c "docker login -u $env:DOCKER_USERNAME --password-stdin < `"$tempFile`""

                            $loginResult = $LASTEXITCODE

                            if ($loginResult -ne 0) {
                                throw "Docker Hub login failed."
                            }

                            Write-Host "Docker Hub login successful."

                        }
                        finally {

                            if (Test-Path $tempFile) {
                                Remove-Item $tempFile -Force -ErrorAction SilentlyContinue
                            }

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

                    powershell '''
                        $ErrorActionPreference = "Stop"

                        Write-Host "========================================"
                        Write-Host "Tagging Docker Hub Image..."
                        Write-Host "========================================"

                        docker tag docker-jenkins-app:latest spk1354/docker-jenkins-app:latest

                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker Hub tag failed."
                        }

                        Write-Host "========================================"
                        Write-Host "Pushing Image to Docker Hub..."
                        Write-Host "========================================"

                        docker push spk1354/docker-jenkins-app:latest

                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker Hub push failed."
                        }

                        Write-Host "Docker Hub push successful."
                    '''
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

                        Write-Host "========================================"
                        Write-Host "Amazon ECR Authentication"
                        Write-Host "========================================"

                        $registry = "018913575233.dkr.ecr.ap-southeast-2.amazonaws.com"
                        $repository = "docker-jenkins-app"
                        $image = "$registry/$repository:latest"

                        $env:AWS_CLI_AUTO_PROMPT = "off"

                        # Temporary Docker configuration directory
                        $config = Join-Path $env:WORKSPACE ".docker-ecr"

                        try {

                            # Remove old configuration if it exists
                            if (Test-Path $config) {
                                Remove-Item $config -Recurse -Force
                            }

                            New-Item -ItemType Directory -Path $config -Force | Out-Null

                            Write-Host "========================================"
                            Write-Host "Getting ECR authentication token..."
                            Write-Host "========================================"

                            $password = aws ecr get-login-password --region ap-southeast-2

                            if ($LASTEXITCODE -ne 0) {
                                throw "AWS ECR get-login-password failed."
                            }

                            if ([string]::IsNullOrWhiteSpace($password)) {
                                throw "ECR authentication password is empty."
                            }

                            Write-Host "ECR authentication token received."

                            # Create AWS ECR authentication string
                            $authString = "AWS:" + $password

                            $authBytes = [System.Text.Encoding]::UTF8.GetBytes($authString)

                            $authBase64 = [System.Convert]::ToBase64String($authBytes)

                            # Create Docker config.json
                            $dockerConfig = @{
                                auths = @{
                                    $registry = @{
                                        auth = $authBase64
                                    }
                                }
                            }

                            $configFile = Join-Path $config "config.json"

                            $dockerConfig |
                                ConvertTo-Json -Depth 10 |
                                Set-Content -Path $configFile -Encoding ascii

                            Write-Host "========================================"
                            Write-Host "Docker ECR configuration created."
                            Write-Host "========================================"

                            # Tag image for ECR
                            docker --config $config tag `
                                docker-jenkins-app:latest `
                                $image

                            if ($LASTEXITCODE -ne 0) {
                                throw "Docker ECR tag failed."
                            }

                            Write-Host "ECR image tagged successfully."

                            Write-Host "========================================"
                            Write-Host "Pushing Image to Amazon ECR..."
                            Write-Host "========================================"

                            # Push using temporary Docker configuration
                            docker --config $config push $image

                            if ($LASTEXITCODE -ne 0) {
                                throw "Docker ECR push failed."
                            }

                            Write-Host "========================================"
                            Write-Host "Amazon ECR push successful."
                            Write-Host "========================================"

                        }
                        finally {

                            # Always remove temporary ECR credentials
                            if (Test-Path $config) {
                                Remove-Item $config -Recurse -Force -ErrorAction SilentlyContinue
                            }

                            Write-Host "Temporary ECR Docker configuration removed."
                        }
                    '''
                }
            }
        }


        stage('Pull Image from Docker Hub') {
            steps {
                powershell '''
                    $ErrorActionPreference = "Stop"

                    Write-Host "========================================"
                    Write-Host "Pulling Image from Docker Hub..."
                    Write-Host "========================================"

                    docker pull spk1354/docker-jenkins-app:latest

                    if ($LASTEXITCODE -ne 0) {
                        throw "Docker Hub pull failed."
                    }

                    Write-Host "Docker Hub image pulled successfully."
                '''
            }
        }


        stage('Run Docker Hub Image') {
            steps {
                powershell '''
                    $ErrorActionPreference = "Stop"

                    Write-Host "========================================"
                    Write-Host "Running Docker Hub Image..."
                    Write-Host "========================================"

                    docker stop docker-jenkins-app-jenkins 2>$null
                    docker rm docker-jenkins-app-jenkins 2>$null

                    docker run -d `
                        -p 8085:80 `
                        --name docker-jenkins-app-jenkins `
                        spk1354/docker-jenkins-app:latest

                    if ($LASTEXITCODE -ne 0) {
                        throw "Docker container run failed."
                    }

                    Write-Host "Container started successfully."
                    Write-Host "Application URL: http://localhost:8085"
                '''
            }
        }


        stage('Docker Container Check') {
            steps {
                powershell '''
                    Write-Host "========================================"
                    Write-Host "Checking Docker Container..."
                    Write-Host "========================================"

                    docker ps

                    docker inspect docker-jenkins-app-jenkins

                    if ($LASTEXITCODE -ne 0) {
                        throw "Docker container check failed."
                    }
                '''
            }
        }


        stage('Volume Persistence Test') {
            steps {
                powershell '''
                    $ErrorActionPreference = "Stop"

                    Write-Host "========================================"
                    Write-Host "Testing Docker Volume Persistence..."
                    Write-Host "========================================"

                    docker stop docker-jenkins-app 2>$null
                    docker rm docker-jenkins-app 2>$null

                    docker volume create jenkins-docker-volume

                    docker run -d `
                        -p 8081:80 `
                        --name docker-jenkins-app `
                        -v jenkins-docker-volume:/usr/share/nginx/html `
                        nginx

                    if ($LASTEXITCODE -ne 0) {
                        throw "Docker volume container creation failed."
                    }

                    docker exec docker-jenkins-app `
                        sh -c "echo 'Volume Persistence Test' > /usr/share/nginx/html/volume-test.txt"

                    Write-Host "Checking volume data..."

                    docker exec docker-jenkins-app `
                        cat /usr/share/nginx/html/volume-test.txt

                    Write-Host "Stopping container..."

                    docker stop docker-jenkins-app

                    Write-Host "Removing container..."

                    docker rm docker-jenkins-app

                    Write-Host "Recreating container using same volume..."

                    docker run -d `
                        -p 8081:80 `
                        --name docker-jenkins-app `
                        -v jenkins-docker-volume:/usr/share/nginx/html `
                        nginx

                    if ($LASTEXITCODE -ne 0) {
                        throw "Docker volume recreation failed."
                    }

                    Write-Host "Checking persisted data..."

                    docker exec docker-jenkins-app `
                        cat /usr/share/nginx/html/volume-test.txt

                    Write-Host "========================================"
                    Write-Host "Volume persistence test successful."
                    Write-Host "========================================"
                '''
            }
        }
    }


    post {

        success {
            echo '========================================'
            echo 'JENKINS PIPELINE COMPLETED SUCCESSFULLY'
            echo '========================================'
        }

        failure {
            echo '========================================'
            echo 'JENKINS PIPELINE FAILED'
            echo 'Check the console output for the failed stage.'
            echo '========================================'
        }

        always {
            powershell '''
                # Remove temporary ECR configuration if anything remains
                $config = Join-Path $env:WORKSPACE ".docker-ecr"

                if (Test-Path $config) {
                    Remove-Item $config -Recurse -Force -ErrorAction SilentlyContinue
                }
            '''
        }
    }
}
