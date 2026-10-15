pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'gopikaks/online-food-ordering-portal'
        SERVICE_NAME = 'foodie-service'
        TEST_PORT = '18082'
        NODE_PORT = '30012'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.IMAGE_TAG = "${BUILD_NUMBER}"
                }
                bat 'docker build -t %DOCKER_IMAGE%:%IMAGE_TAG% .'
            }
        }

        stage('Test Website') {
            steps {
                bat '''
                    docker rm -f foodie-test 2>NUL
                    docker run -d --name foodie-test -p %TEST_PORT%:80 %DOCKER_IMAGE%:%IMAGE_TAG%
                    if errorlevel 1 exit /b 1
                '''
                powershell '''
                    Start-Sleep -Seconds 5
                    try {
                        $response = Invoke-WebRequest -Uri "http://localhost:$env:TEST_PORT/" -UseBasicParsing -TimeoutSec 10
                        if ($response.StatusCode -ne 200) {
                            throw "Website test failed"
                        }
                        Write-Host "FoodieExpress website test passed."
                    } catch {
                        docker logs foodie-test
                        exit 1
                    }
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_TOKEN'
                )]) {
                    bat '''
                        docker login -u "%DH_USER%" -p "%DH_TOKEN%"
                        if errorlevel 1 exit /b 1
                        docker push %DOCKER_IMAGE%:%IMAGE_TAG%
                        if errorlevel 1 exit /b 1
                        docker tag %DOCKER_IMAGE%:%IMAGE_TAG% %DOCKER_IMAGE%:latest
                        docker push %DOCKER_IMAGE%:latest
                        if errorlevel 1 exit /b 1
                        docker logout
                    '''
                }
            }
        }

        stage('Identify Blue-Green Target') {
            steps {
                withCredentials([file(
                    credentialsId: 'kuberconfig',
                    variable: 'KUBECONFIG_FILE'
                )]) {
                    script {
                        def activeSlot = bat(
                            script: '@echo off\r\nset "KUBECONFIG=%KUBECONFIG_FILE%"\r\nkubectl get service foodie-service -o jsonpath={.spec.selector.slot}',
                            returnStdout: true
                        ).trim()

                        if (activeSlot.endsWith('blue')) {
                            activeSlot = 'blue'
                        } else if (activeSlot.endsWith('green')) {
                            activeSlot = 'green'
                        } else {
                            error("Could not detect active slot: ${activeSlot}")
                        }

                        env.ACTIVE_SLOT = activeSlot
                        env.NEXT_SLOT = activeSlot == 'blue' ? 'green' : 'blue'

                        echo "Current active slot: ${env.ACTIVE_SLOT}"
                        echo "New release target: ${env.NEXT_SLOT}"
                    }
                }
            }
        }

        stage('Deploy to Inactive Environment') {
            steps {
                withCredentials([file(
                    credentialsId: 'kuberconfig',
                    variable: 'KUBECONFIG_FILE'
                )]) {
                    bat '''
                        @echo off
                        set "KUBECONFIG=%KUBECONFIG_FILE%"
                        kubectl set image deployment/foodie-%NEXT_SLOT% foodie=%DOCKER_IMAGE%:%IMAGE_TAG%
                        if errorlevel 1 exit /b 1
                        kubectl rollout status deployment/foodie-%NEXT_SLOT% --timeout=180s
                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }

        stage('Switch Traffic') {
            steps {
                withCredentials([file(
                    credentialsId: 'kuberconfig',
                    variable: 'KUBECONFIG_FILE'
                )]) {
                    bat '''
                        @echo off
                        set "KUBECONFIG=%KUBECONFIG_FILE%"
                        kubectl patch service %SERVICE_NAME% -p "{\\"spec\\":{\\"selector\\":{\\"app\\":\\"foodie\\",\\"slot\\":\\"%NEXT_SLOT%\\"}}}"
                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                powershell '''
                    Start-Sleep -Seconds 5
                    try {
                        $response = Invoke-WebRequest -Uri "http://localhost:$env:NODE_PORT/" -UseBasicParsing -TimeoutSec 15
                        if ($response.StatusCode -ne 200) {
                            throw "Deployment verification failed"
                        }
                        Write-Host "Food Ordering Portal is responding successfully."
                    } catch {
                        Write-Host "Verification failed. Rolling traffic back."
                        kubectl --kubeconfig="$env:KUBECONFIG_FILE" patch service foodie-service -p "{\\"spec\\":{\\"selector\\":{\\"app\\":\\"foodie\\",\\"slot\\":\\"$env:ACTIVE_SLOT\\"}}}"
                        exit 1
                    }
                '''
            }
        }
    }

    post {
        always {
            bat 'docker rm -f foodie-test 2>NUL & exit /b 0'
        }

        success {
            echo 'Blue-green deployment completed successfully.'
            echo 'Application URL: http://localhost:30012'
        }

        failure {
            echo 'Pipeline failed. Check Console Output for the failing stage.'
        }
    }
}
