pipeline {
    agent any

    stages {

        stage('Docker Check') {
            steps {
                bat 'docker --version'
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '''
                    cd /d C:\\docker-jenkins-project
                    docker build -t docker-jenkins-app:latest .
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
                            throw "Docker Hub login failed."
                        }

                        Write-Host "Docker Hub login successful."
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

                    bat '''
                        docker tag docker-jenkins-app:latest %DOCKER_USERNAME%/docker-jenkins-app:latest

                        docker push %DOCKER_USERNAME%/docker-jenkins-app:latest
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

                        # ========================================
                        # AWS ECR DETAILS
                        # ========================================

                        $registry = "018913575233.dkr.ecr.ap-southeast-2.amazonaws.com"

                        $repository = "docker-jenkins-app"

                        $image = "$registry/$repository:latest"

                        # Disable AWS CLI auto prompt
                        $env:AWS_CLI_AUTO_PROMPT = "off"

                        # ========================================
                        # TEMPORARY DOCKER CONFIG
                        # ========================================

                        $config = Join-Path $env:WORKSPACE ".docker-ecr"

                        Write-Host ""
                        Write-Host "========================================"
                        Write-Host "AWS ECR LOGIN AND PUSH"
                        Write-Host "========================================"

                        Write-Host ""
                        Write-Host "Creating temporary Docker configuration..."

                        if (Test-Path $config) {
                            Remove-Item $config -Recurse -Force
                        }

                        New-Item `
                            -ItemType Directory `
                            -Path $config `
                            -Force | Out-Null

                        Write-Host "Temporary Docker configuration created."

                        # ========================================
                        # GET ECR PASSWORD
                        # ========================================

                        Write-Host ""
                        Write-Host "========================================"
                        Write-Host "Getting ECR login password..."
                        Write-Host "========================================"

                        $password = aws ecr get-login-password `
                            --region ap-southeast-2

                        if ($LASTEXITCODE -ne 0) {
                            throw "AWS ECR get-login-password failed."
                        }

                        if ([string]::IsNullOrWhiteSpace($password)) {
                            throw "ECR login password is empty."
                        }

                        Write-Host "ECR login password received successfully."

                        # ========================================
                        # CREATE DOCKER AUTH CONFIG
                        # ========================================

                        Write-Host ""
                        Write-Host "========================================"
                        Write-Host "Creating Docker ECR authentication..."
                        Write-Host "========================================"

                        $authString = "AWS:" + $password

                        $authBytes = [System.Text.Encoding]::UTF8.GetBytes(
                            $authString
                        )

                        $authBase64 = [System.Convert]::ToBase64String(
                            $authBytes
                        )

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
                            Set-Content `
                                -Path $configFile `
                                -Encoding ascii

                        Write-Host "Docker ECR authentication created."

                        # ========================================
                        # TAG IMAGE FOR ECR
                        # ========================================

                        Write-Host ""
                        Write-Host "========================================"
                        Write-Host "Tagging Docker image for ECR..."
                        Write-Host "========================================"

                        docker --config $config tag `
                            docker-jenkins-app:latest `
                            $image

                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker ECR tag failed."
                        }

                        Write-Host "Docker image tagged successfully."

                        Write-Host ""
                        Write-Host "ECR image:"
                        Write-Host $image

                        # ========================================
                        # PUSH IMAGE TO ECR
                        # ========================================

                        Write-Host ""
                        Write-Host "========================================"
                        Write-Host "Pushing image to Amazon ECR..."
                        Write-Host "========================================"

                        docker --config $config push $image

                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker ECR push failed."
                        }

                        Write-Host ""
                        Write-Host "========================================"
                        Write-Host "ECR PUSH SUCCESSFUL"
                        Write-Host "========================================"

                        Write-Host ""
                        Write-Host "Image pushed:"
                        Write-Host $image

                        # ========================================
                        # CLEAN TEMPORARY CREDENTIALS
                        # ========================================

                        Write-Host ""
                        Write-Host "Removing temporary Docker credentials..."

                        if (Test-Path $config) {
                            Remove-Item $config -Recurse -Force
                        }

                        Write-Host "Temporary Docker credentials removed."

                        Write-Host ""
                        Write-Host "========================================"
                        Write-Host "ECR STAGE COMPLETED SUCCESSFULLY"
                        Write-Host "========================================"
                    '''
                }
            }
        }

        stage('Pull Image from Docker Hub') {
            steps {

                bat '''
                    docker pull spk1354/docker-jenkins-app:latest
                '''
            }
        }

        stage('Run Docker Hub Image') {
            steps {

                bat '''
                    docker rm -f docker-jenkins-app-jenkins 2>NUL || echo No old test container found
                '''

                bat '''
                    docker run -d -p 8084:80 --name docker-jenkins-app-jenkins spk1354/docker-jenkins-app:latest
                '''
            }
        }

        stage('Docker Container Check') {
            steps {

                bat '''
                    docker ps
                '''

                bat '''
                    docker inspect docker-jenkins-app-jenkins
                '''
            }
        }

        stage('Volume Persistence Test') {
            steps {

                bat '''
                    docker exec docker-jenkins-app cat /usr/share/nginx/html/volume-test.txt
                '''

                bat '''
                    docker stop docker-jenkins-app
                '''

                bat '''
                    docker rm docker-jenkins-app
                '''

                bat '''
                    docker run -d -p 8081:80 --name docker-jenkins-app -v jenkins-docker-volume:/usr/share/nginx/html nginx
                '''

                bat '''
                    docker exec docker-jenkins-app cat /usr/share/nginx/html/volume-test.txt
                '''
            }
        }
    }

    post {

        success {
            echo ''
            echo '========================================'
            echo 'JENKINS PIPELINE COMPLETED SUCCESSFULLY'
            echo '========================================'
        }

        failure {
            echo ''
            echo '========================================'
            echo 'JENKINS PIPELINE FAILED'
            echo '========================================'
            echo 'Please check the console output.'
        }

        always {
            echo ''
            echo 'Pipeline execution completed.'
        }
    }
}
