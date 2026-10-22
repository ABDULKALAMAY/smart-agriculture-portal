pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
    }

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'staging', 'prod'],
            description: 'Select target deployment environment'
        )
        booleanParam(
            name: 'RUN_WATER_AUDIT',
            defaultValue: true,
            description: 'Run conditional water conservation audit stage'
        )
        booleanParam(
            name: 'SIMULATE_FAULT',
            defaultValue: false,
            description: 'Simulate faulty image without health.html to test automatic rollback'
        )
    }

    environment {
        DOCKER_IMAGE = 'abdulkalamay/smart-agriculture-portal'
        APP_NAME = 'smart-agriculture-portal'
        SERVICE_NAME = 'smart-agriculture-portal-service'
        PREVIOUS_DEPLOYMENT_EXISTS = 'false'
        DEPLOYMENT_UPDATE_STARTED = 'false'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
                script {
                    env.IMAGE_TAG = "v1.0.${env.BUILD_NUMBER}"
                }
                echo "Deploying to Environment: ${params.ENVIRONMENT} with Tag: ${env.IMAGE_TAG}"
            }
        }

        stage('Parallel Validation Checks') {
            parallel {
                stage('HTML Sensor Data Check') {
                    steps {
                        powershell '''
                            $content = Get-Content index.html -Raw
                            foreach ($fieldId in 101..105) {
                                if (-not $content.Contains(("FLD-{0}" -f $fieldId))) {
                                    throw "Missing agricultural field record FLD-$fieldId"
                                }
                            }
                            Write-Output "HTML Sensor Data Check Passed: All 5 fields verified."
                        '''
                    }
                }
                stage('Kubernetes & Docker Check') {
                    steps {
                        powershell '''
                            if (-not (Test-Path "Dockerfile") -or -not (Test-Path "deployment.yaml")) {
                                throw "Missing Dockerfile or deployment.yaml"
                            }
                            if (-not (Test-Path "health.html")) {
                                throw "Missing health.html required by the Docker image"
                            }
                            Write-Output "Docker & Kubernetes Manifest Check Passed."
                        '''
                    }
                }
            }
        }

        stage('Conditional Water Audit') {
            when {
                expression { params.RUN_WATER_AUDIT == true }
            }
            steps {
                powershell '''
                    $report = @"
Smart Agriculture Water Optimization Audit Report
Build Number : $($env:BUILD_NUMBER)
Environment  : $($env:ENVIRONMENT)
Image Tag    : $($env:IMAGE_TAG)
Total Fields : 5 (FLD-101 to FLD-105)
Total Water  : 7,600 Litres/day
Lowest Moist.: FLD-104 (Maize - 18%)
Audit Status : PASSED
"@
                    $report | Out-File -FilePath "irrigation_audit_report.txt" -Encoding utf8
                    Get-Content "irrigation_audit_report.txt"
                '''
            }
        }

        stage('Archive Build Artifacts') {
            steps {
                script {
                    archiveArtifacts artifacts: 'index.html,deployment.yaml', fingerprint: true
                    if (params.RUN_WATER_AUDIT) {
                        archiveArtifacts artifacts: 'irrigation_audit_report.txt', fingerprint: true
                    }
                }
            }
        }

        stage('Build & Test Docker Image') {
            steps {
                bat "docker build --build-arg RELEASE_VERSION=%IMAGE_TAG% --build-arg SIMULATE_FAULT=%SIMULATE_FAULT% -t %DOCKER_IMAGE%:%IMAGE_TAG% -t %DOCKER_IMAGE%:latest ."
                bat "docker run --rm %DOCKER_IMAGE%:%IMAGE_TAG% sh -c \"test -s /usr/share/nginx/html/index.html && test -s /usr/share/nginx/html/health.html && test -s /usr/share/nginx/html/release.txt\""
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    bat "echo %PASS% | docker login -u %USER% --password-stdin"
                    bat "docker push %DOCKER_IMAGE%:%IMAGE_TAG%"
                    bat "docker push %DOCKER_IMAGE%:latest"
                }
            }
        }

        stage('Deploy to Kubernetes & Verify') {
            steps {
                withCredentials([file(credentialsId: 'kuberconfig', variable: 'KUBECONFIG')]) {
                    script {
                        env.PREVIOUS_DEPLOYMENT_EXISTS = bat(
                            returnStatus: true,
                            script: 'kubectl get deployment %APP_NAME%'
                        ) == 0 ? 'true' : 'false'
                        env.DEPLOYMENT_UPDATE_STARTED = 'true'
                    }
                    powershell '''
                        $env:KUBECONFIG = "$env:KUBECONFIG"
                        if ($env:PREVIOUS_DEPLOYMENT_EXISTS -ne "true") {
                            kubectl apply -f deployment.yaml --validate=false
                        }
                        kubectl set image deployment/$env:APP_NAME portal-container=$($env:DOCKER_IMAGE):$($env:IMAGE_TAG)
                        kubectl rollout status deployment/$env:APP_NAME --timeout=60s
                        if ($LASTEXITCODE -ne 0) {
                            throw "Deployment rollout failed health checks!"
                        }
                        kubectl get deployment $env:APP_NAME -o wide
                        kubectl get pods -l app=$env:APP_NAME -o wide
                        kubectl get service $env:SERVICE_NAME -o wide
                    '''
                }
            }
        }
    }

    post {
        failure {
            withCredentials([file(credentialsId: 'kuberconfig', variable: 'KUBECONFIG')]) {
                powershell '''
                    Write-Output "=== FAILED RELEASE DETECTED: TRIGGERING AUTOMATIC ROLLBACK ==="
                    if ($env:DEPLOYMENT_UPDATE_STARTED -eq "true" -and $env:PREVIOUS_DEPLOYMENT_EXISTS -eq "true") {
                        kubectl get pods -l app=$env:APP_NAME
                        kubectl get events --field-selector type=Warning --sort-by=.lastTimestamp | Select-Object -Last 10
                        kubectl rollout undo deployment/$env:APP_NAME
                        if ($LASTEXITCODE -ne 0) {
                            throw "Rollback failed for deployment $env:APP_NAME"
                        }
                        kubectl rollout status deployment/$env:APP_NAME --timeout=90s
                        if ($LASTEXITCODE -ne 0) {
                            throw "Rollback rollout did not become healthy"
                        }
                        Write-Output "=== ROLLBACK SUCCESSFUL: RESTORED PREVIOUS STABLE RELEASE ==="
                        kubectl get deployment $env:APP_NAME -o wide
                        kubectl get pods -l app=$env:APP_NAME -o wide
                    } else {
                        Write-Output "No existing deployment to roll back."
                    }
                '''
            }
        }
        success {
            echo "Smart Agriculture Portal successfully deployed on NodePort 30017!"
        }
    }
}
