pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_NAME = 'gopikaks/online-food-ordering-portal'
        SERVICE_NAME = 'foodie-service'
        PREVIOUS_SLOT = ''
        TARGET_SLOT = ''
        TRAFFIC_SWITCHED = 'false'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build -t %IMAGE_NAME%:%BUILD_NUMBER% .'
            }
        }

        stage('Test Website') {
            steps {
                bat '''
                    docker rm -f foodie-test-%BUILD_NUMBER% 2>NUL
                    docker run -d --name foodie-test-%BUILD_NUMBER% -p 18082:80 %IMAGE_NAME%:%BUILD_NUMBER%
                    if errorlevel 1 exit /b 1
                '''

                powershell '''
                    $ready = $false

                    for ($i = 0; $i -lt 15; $i++) {
                        try {
                            $response = Invoke-WebRequest `
                                -Uri "http://localhost:18082/" `
                                -TimeoutSec 3 `
                                -UseBasicParsing

                            if ($response.StatusCode -eq 200) {
                                $ready = $true
                                break
                            }
                        }
                        catch {
                            Start-Sleep -Seconds 2
                        }
                    }

                    if (-not $ready) {
                        throw "FoodieExpress website test failed."
                    }

                    Write-Host "FoodieExpress website test passed."
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    bat '''
                        docker login -u "%DH_USER%" -p "%DH_TOKEN%"
                        if errorlevel 1 exit /b 1

                        docker push %IMAGE_NAME%:%BUILD_NUMBER%
                        if errorlevel 1 exit /b 1

                        docker tag %IMAGE_NAME%:%BUILD_NUMBER% %IMAGE_NAME%:latest
                        if errorlevel 1 exit /b 1

                        docker push %IMAGE_NAME%:latest
                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }

        stage('Identify Blue-Green Target') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kuberconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    script {
                        def active = bat(
                            returnStdout: true,
                            script: '''
                                @echo off
                                kubectl --kubeconfig="%KUBECONFIG_FILE%" get service foodie-service -o "jsonpath={.spec.selector.slot}"
                            '''
                        ).trim()

                        if (active != 'blue' && active != 'green') {
                            error("Unexpected active slot: ${active}")
                        }

                        env.PREVIOUS_SLOT = active
                        env.TARGET_SLOT = active == 'blue' ? 'green' : 'blue'

                        echo "Current active slot: ${env.PREVIOUS_SLOT}"
                        echo "New release target: ${env.TARGET_SLOT}"
                    }
                }
            }
        }

        stage('Deploy to Inactive Environment') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kuberconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    bat '''
                        kubectl --kubeconfig="%KUBECONFIG_FILE%" set image deployment/foodie-%TARGET_SLOT% foodie=%IMAGE_NAME%:%BUILD_NUMBER%
                        if errorlevel 1 exit /b 1

                        kubectl --kubeconfig="%KUBECONFIG_FILE%" rollout status deployment/foodie-%TARGET_SLOT% --timeout=180s
                        if errorlevel 1 exit /b 1
                    '''
                }
            }
        }

        stage('Switch Traffic') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kuberconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    bat '''
                        kubectl --kubeconfig="%KUBECONFIG_FILE%" patch service foodie-service --type=merge -p "{\\"spec\\":{\\"selector\\":{\\"app\\":\\"foodie\\",\\"slot\\":\\"%TARGET_SLOT%\\"}}}"
                        if errorlevel 1 exit /b 1
                    '''

                    script {
                        env.TRAFFIC_SWITCHED = 'true'
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kuberconfig',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    bat '''
                        kubectl --kubeconfig="%KUBECONFIG_FILE%" rollout status deployment/foodie-%TARGET_SLOT% --timeout=180s
                        if errorlevel 1 exit /b 1

                        kubectl --kubeconfig="%KUBECONFIG_FILE%" get deployments
                        if errorlevel 1 exit /b 1

                        kubectl --kubeconfig="%KUBECONFIG_FILE%" get pods -l app=foodie
                        if errorlevel 1 exit /b 1

                        kubectl --kubeconfig="%KUBECONFIG_FILE%" get service foodie-service
                        if errorlevel 1 exit /b 1
                    '''

                    powershell '''
                        $ready = $false

                        for ($i = 0; $i -lt 12; $i++) {
                            try {
                                $response = Invoke-WebRequest `
                                    -Uri "http://localhost:30012/" `
                                    -TimeoutSec 5 `
                                    -UseBasicParsing

                                if ($response.StatusCode -eq 200) {
                                    $ready = $true
                                    break
                                }
                            }
                            catch {
                                Start-Sleep -Seconds 3
                            }
                        }

                        if (-not $ready) {
                            throw "FoodieExpress health check failed."
                        }

                        Write-Host "FoodieExpress deployment verified."
                    '''
                }
            }
        }
    }

    post {
        always {
            bat 'docker rm -f foodie-test-%BUILD_NUMBER% 2>NUL'
        }

        success {
            echo 'FoodieExpress CI/CD and blue-green release completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Checking whether traffic needs to be restored.'

            script {
                if (env.TRAFFIC_SWITCHED == 'true' &&
                    (env.PREVIOUS_SLOT == 'blue' ||
                     env.PREVIOUS_SLOT == 'green')) {

                    withCredentials([
                        file(
                            credentialsId: 'kuberconfig',
                            variable: 'KUBECONFIG_FILE'
                        )
                    ]) {
                        bat '''
                            kubectl --kubeconfig="%KUBECONFIG_FILE%" patch service foodie-service --type=merge -p "{\\"spec\\":{\\"selector\\":{\\"app\\":\\"foodie\\",\\"slot\\":\\"%PREVIOUS_SLOT%\\"}}}"
                            if errorlevel 1 exit /b 1
                        '''

                        echo 'Traffic restoration to the previous slot was requested.'
                    }
                } else {
                    echo 'Traffic was not switched; the existing active environment was left unchanged.'
                }
            }
        }
    }
}
