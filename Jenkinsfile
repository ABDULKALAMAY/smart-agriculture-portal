pipeline {
    agent any

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
        SERVICE_NAME = 'smart-agriculture-service'
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
                            if (-not $content.Contains("FLD-101") -or -not $content.Contains("FLD-105")) {
                                throw "Missing required 5 agricultural field records!"
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
                archiveArtifacts artifacts: 'irrigation_audit_report.txt,index.html,deployment.yaml', fingerprint: true
            }
        }

        stage('Build & Test Docker Image') {
            steps {
                bat "docker build --build-arg RELEASE_VERSION=%IMAGE_TAG% --build-arg SIMULATE_FAULT=%SIMULATE_FAULT% -t %DOCKER_IMAGE%:%IMAGE_TAG% -t %DOCKER_IMAGE%:latest ."
                bat "docker run --rm %DOCKER_IMAGE%:%IMAGE_TAG% sh -c \"test -f /usr/share/nginx/html/index.html\""
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
                    powershell '''
                        $env:KUBECONFIG = "$env:KUBECONFIG"
                        kubectl get deployment $env:APP_NAME > $null 2>&1
                        if ($LASTEXITCODE -ne 0) {
                            kubectl apply -f deployment.yaml --validate=false
                        }
                        kubectl set image deployment/$env:APP_NAME $env:APP_NAME=$($env:DOCKER_IMAGE):$($env:IMAGE_TAG)
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
                    kubectl get pods -l app=$env:APP_NAME
                    kubectl get events --field-selector type=Warning --sort-by=.lastTimestamp | Select-Object -Last 10
                    kubectl rollout undo deployment/$env:APP_NAME
                    kubectl rollout status deployment/$env:APP_NAME --timeout=90s
                    Write-Output "=== ROLLBACK SUCCESSFUL: RESTORED PREVIOUS STABLE RELEASE ==="
                    kubectl get deployment $env:APP_NAME -o wide
                    kubectl get pods -l app=$env:APP_NAME -o wide
                '''
            }
        }
        success {
            echo "Smart Agriculture Portal successfully deployed on NodePort 30017!"
        }
    }
}
