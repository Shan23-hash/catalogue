pipeline {

    agent {
        label 'AGENT-1'
    }

    environment {
        REGION = 'us-east-1'
        ACC_ID = '565257597039'
        PROJECT = 'roboshop'
        COMPONENT = 'catalogue'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    parameters {
        booleanParam(
            name: 'deploy',
            defaultValue: false,
            description: 'Deploy to DEV after security scan passes'
        )
    }

    stages {

        stage('Read package.json') {
            steps {
                script {
                    def packageJson = readJSON file: 'package.json'
                    def version = packageJson.version

                    echo "======================================"
                    echo "Reading package.json"
                    echo "======================================"

                    echo "Package name    : ${packageJson.name}"
                    echo "Package version : ${version}"

                    if (version == null || version.toString().trim() == '') {
                        error '❌ package.json version is missing'
                    }

                    writeFile(
                        file: 'app-version.txt',
                        text: version.toString().trim()
                    )

                    echo "✅ APP_VERSION saved: ${version}"
                }
            }
        }

        stage('Verify Version') {
            steps {
                script {
                    def appVersion = readFile('app-version.txt').trim()

                    echo "======================================"
                    echo "Verify Application Version"
                    echo "======================================"

                    echo "APP_VERSION: [${appVersion}]"

                    if (!appVersion || appVersion == 'null') {
                        error "❌ Invalid APP_VERSION: [${appVersion}]"
                    }

                    echo "✅ Version validation successful"
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    echo "======================================"
                    echo "Installing Dependencies"
                    echo "======================================"

                    npm install
                '''
            }
        }

        stage('Unit Testing') {
            steps {
                sh '''
                    echo "======================================"
                    echo "Unit Testing"
                    echo "======================================"

                    echo "unit tests"
                '''
            }
        }

        /*
        stage('Sonar Scan') {
            environment {
                scannerHome = tool 'sonar-7.2'
            }
            steps {
                script {
                    withSonarQubeEnv(installationName: 'sonar-7.2') {
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        */

        stage('Check Dependabot Alerts') {
            environment {
                GITHUB_TOKEN = credentials('github-token')
            }

            steps {
                script {

                    def response = sh(
                        script: '''
                            curl -s \
                              -H "Accept: application/vnd.github+json" \
                              -H "Authorization: Bearer $GITHUB_TOKEN" \
                              https://api.github.com/repos/Shan23-hash/catalogue/dependabot/alerts
                        ''',
                        returnStdout: true
                    ).trim()

                    def json = readJSON text: response

                    def criticalOrHigh = json.findAll { alert ->

                        def severity =
                            alert?.security_advisory?.severity?.toLowerCase()

                        def state =
                            alert?.state?.toLowerCase()

                        return (
                            state == 'open' &&
                            (severity == 'critical' || severity == 'high')
                        )
                    }

                    if (criticalOrHigh.size() > 0) {

                        error(
                            "❌ Found ${criticalOrHigh.size()} HIGH/CRITICAL Dependabot alerts. Failing pipeline!"
                        )

                    } else {

                        echo "✅ No HIGH/CRITICAL Dependabot alerts found."
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {

                    def appVersion =
                        readFile('app-version.txt').trim()

                    echo "======================================"
                    echo "Docker Build"
                    echo "======================================"

                    echo "APP_VERSION: [${appVersion}]"

                    withAWS(
                        credentials: 'aws-creds',
                        region: 'us-east-1'
                    ) {

                        sh """
                            echo "======================================"
                            echo "Login to Amazon ECR"
                            echo "======================================"

                            aws ecr get-login-password \
                              --region ${REGION} | \
                            docker login \
                              --username AWS \
                              --password-stdin \
                              ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com

                            echo "======================================"
                            echo "Building Docker Image"
                            echo "======================================"

                            docker buildx build \
                              --provenance=false \
                              --sbom=false \
                              --load \
                              -t ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}/${COMPONENT}:${appVersion} \
                              .

                            echo "======================================"
                            echo "Pushing Docker Image"
                            echo "======================================"

                            docker push \
                              ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}/${COMPONENT}:${appVersion}

                            echo "======================================"
                            echo "Docker Image Push Successful"
                            echo "======================================"

                            echo "Image:"
                            echo "${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}/${COMPONENT}:${appVersion}"
                        """
                    }
                }
            }
        }

        stage('Check Scan Results') {
            steps {
                script {

                    def appVersion =
                        readFile('app-version.txt').trim()

                    withAWS(
                        credentials: 'aws-creds',
                        region: 'us-east-1'
                    ) {

                        echo "======================================"
                        echo "ECR SECURITY SCAN"
                        echo "======================================"

                        echo "Image: ${COMPONENT}:${appVersion}"

                        def scanStatus = ''

                        timeout(time: 5, unit: 'MINUTES') {

                            waitUntil {

                                def result = sh(
                                    script: """
                                        aws ecr describe-image-scan-findings \
                                          --repository-name ${PROJECT}/${COMPONENT} \
                                          --image-id imageTag=${appVersion} \
                                          --region ${REGION} \
                                          --query 'imageScanStatus.status' \
                                          --output text \
                                          2>/tmp/ecr-scan-error
                                    """,
                                    returnStatus: true
                                )

                                if (result != 0) {

                                    def errorMessage = sh(
                                        script: 'cat /tmp/ecr-scan-error',
                                        returnStdout: true
                                    ).trim()

                                    echo "⏳ ECR scan not available yet..."
                                    echo "AWS response: ${errorMessage}"

                                    sleep 15

                                    return false
                                }

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

                                if (scanStatus == 'COMPLETE') {
                                    return true
                                }

                                if (
                                    scanStatus == 'FAILED' ||
                                    scanStatus == 'UNSUPPORTED_IMAGE' ||
                                    scanStatus == 'FINDINGS_UNAVAILABLE'
                                ) {

                                    error(
                                        "❌ ECR scan failed. Status: ${scanStatus}"
                                    )
                                }

                                echo "⏳ ECR scan still running..."

                                sleep 15

                                return false
                            }
                        }

                        echo "======================================"
                        echo "ECR SCAN COMPLETED"
                        echo "======================================"

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
                        }

                        echo "✅ ECR SECURITY QUALITY GATE PASSED"

                        echo "No HIGH or CRITICAL vulnerabilities found."
                        echo "Deployment is allowed."
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

                    def appVersion =
                        readFile('app-version.txt').trim()

                    echo "======================================"
                    echo "Trigger Catalogue CD"
                    echo "======================================"

                    echo "Application Version: ${appVersion}"
                    echo "Environment: dev"

                    build job: 'catalogue-cd',
                        parameters: [
                            string(
                                name: 'appVersion',
                                value: appVersion
                            ),
                            string(
                                name: 'deploy_to',
                                value: 'dev'
                            )
                        ],
                        propagate: false,
                        wait: false

                    echo "✅ Catalogue CD triggered successfully."
                }
            }
        }
    }

    post {

        always {
            echo "======================================"
            echo "Post Actions"
            echo "======================================"

            echo "I will always say Hello again!"

            deleteDir()
        }

        success {
            echo "======================================"
            echo "Hello Success"
            echo "======================================"
        }

        failure {
            echo "======================================"
            echo "Hello Failure"
            echo "======================================"
        }
    }
}