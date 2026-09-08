pipeline {

    agent {
        label 'AGENT-1'
    }

    environment {
        appVersion = ''
        REGION = "us-east-1"
        ACC_ID = "565257597039"
        PROJECT = "roboshop"
        COMPONENT = "catalogue"
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    parameters {
        booleanParam(
            name: 'deploy',
            defaultValue: false,
            description: 'Toggle this value'
        )
    }

    stages {
        
        stage('Read package.json') {
            steps {
                script {
                    def packageJson = readJSON file: 'package.json'
                    appVersion = packageJson.version
                    echo "Package version: ${appVersion}"
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                script {
                    sh """
                        echo "Installing dependencies..."
                        npm install
                    """
                }
            }
        }

        stage('Unit Testing') {
            steps {
                script {
                    sh """
                        echo "unit tests"
                    """
                }
            }
        }

        /* stage('Sonar Scan') {
            environment {
                scannerHome = tool 'sonar-8.1'
            }
            steps {
                script {
                   // Sonar Server envrionment
                   withSonarQubeEnv(installationName: 'sonar-8.1') {
                         sh "${scannerHome}/bin/sonar-scanner"
                   }
                }
            }
        } */

        // Enable webhook in sonarqube server and wait for results
        /* stage("Quality Gate") {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                waitForQualityGate abortPipeline: true }
            }
        } */

        stage('Check Dependabot Alerts') {
            environment {
                GITHUB_TOKEN = credentials('github-token')
            }
            steps {
                script {
                    def response = sh(
                        script: """
                            curl -s \
                            -H "Accept: application/vnd.github+json" \
                            -H "Authorization: token ${GITHUB_TOKEN}" \
                            https://api.github.com/repos/kranthikumar96/catalogue/dependabot/alerts
                        """,
                        returnStdout: true
                    ).trim()
                    def json = readJSON text: response
                    def criticalOrHigh = json.findAll { alert ->
                        def severity =
                            alert?.security_advisory?.severity?.toLowerCase()
                        def state =
                            alert?.state?.toLowerCase()
                        return (
                            state == "open" &&
                            (severity == "critical" || severity == "high")
                        )
                    }
                    if (criticalOrHigh.size() > 0) {
                        error """
                        ❌ Found ${criticalOrHigh.size()}
                        HIGH/CRITICAL Dependabot alerts.

                        Pipeline stopped.
                        """
                    } else {
                        echo "✅ No HIGH/CRITICAL Dependabot alerts found."
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    withAWS(
                        credentials: 'aws-creds',
                        region: "${REGION}"
                    ) {
                        sh """
                            echo "Logging into Amazon ECR..."

                            aws ecr get-login-password \
                                --region ${REGION} | \
                            docker login \
                                --username AWS \
                                --password-stdin \
                                ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com

                            echo "Building Docker image..."
                            docker buildx build \
                                --provenance=false \
                                --load \
                                -t ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}/${COMPONENT}:${appVersion} .
                            echo "Pushing Docker image..."
                            docker push \
                                ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}/${COMPONENT}:${appVersion}
                        """
                    }
                }
            }
        }

        stage('Check ECR Scan Results') {
            steps {
                script {
                    withAWS(
                        credentials: 'aws-creds',
                        region: "${REGION}"
                    ) {
                        echo "Waiting for ECR scan to complete..."
                        def scanStatus = ""
                        timeout(time: 5, unit: 'MINUTES') {
                            waitUntil {
                                scanStatus = sh(
                                    script: """
                                        aws ecr describe-image-scan-findings \
                                            --repository-name ${PROJECT}/${COMPONENT} \
                                            --image-id imageTag=${appVersion} \
                                            --region ${REGION} \
                                            --query 'imageScanStatus.status' \
                                            --output text
                                    """,
                                    returnStdout: true
                                ).trim()
                                echo "ECR Scan Status: ${scanStatus}"
                                if (scanStatus == "COMPLETE") {
                                    return true
                                }
                                if (
                                    scanStatus == "FAILED" ||
                                    scanStatus == "UNSUPPORTED_IMAGE" ||
                                    scanStatus == "FINDINGS_UNAVAILABLE"
                                ) {
                                    error """
                                    ❌ ECR scan failed.
                                    Scan Status:
                                    ${scanStatus}
                                    """
                                }
                                sleep 10
                                return false
                            }
                        }
                        echo "✅ ECR scan completed."
                        def findings = sh(
                            script: """
                                aws ecr describe-image-scan-findings \
                                    --repository-name ${PROJECT}/${COMPONENT} \
                                    --image-id imageTag=${appVersion} \
                                    --region ${REGION} \
                                    --output json
                            """,
                            returnStdout: true
                        ).trim()
                        def json = readJSON text: findings
                        def severityCounts =
                            json.imageScanFindings.findingSeverityCounts ?: [:]
                        def criticalCount =
                            severityCounts.CRITICAL ?: 0
                        def highCount =
                            severityCounts.HIGH ?: 0
                        def mediumCount =
                            severityCounts.MEDIUM ?: 0
                        def lowCount =
                            severityCounts.LOW ?: 0
                        echo "=========================================="
                        echo "       ECR SECURITY QUALITY GATE"
                        echo "=========================================="
                        echo "Image    : ${COMPONENT}:${appVersion}"
                        echo "CRITICAL : ${criticalCount}"
                        echo "HIGH     : ${highCount}"
                        echo "MEDIUM   : ${mediumCount}"
                        echo "LOW      : ${lowCount}"
                        echo "=========================================="
                        if (
                            criticalCount > 0 ||
                            highCount > 0
                        ) {
                            error """
                            ❌ ECR SECURITY QUALITY GATE FAILED
                            Image:
                            ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}/${COMPONENT}:${appVersion}
                            CRITICAL: ${criticalCount}
                            HIGH:     ${highCount}
                            Deployment BLOCKED.
                            """
                        } else {
                            echo "✅ ECR SECURITY QUALITY GATE PASSED"
                            echo """
                            No HIGH or CRITICAL vulnerabilities found.
                            Deployment is allowed.
                            """
                        }
                    }
                }
            }
        }

        stage('Trigger Deploy') {
            when {
                expression {
                    params.deploy
                }
            }
            steps {
                script {
                    build job: 'catalogue-cd',
                    parameters: [
                        string(
                            name: 'appVersion',
                            value: "${appVersion}"
                        ),
                        string(
                            name: 'deploy_to',
                            value: 'dev'
                        )
                    ],
                    propagate: false,
                    wait: false
                }
            }
        }
    }

    post {
        always {
            echo 'I will always say Hello again!'
            deleteDir()
        }
        success {
            echo 'Hello Success'
        }
        failure {
            echo 'Hello Failure'
        }
    }
}