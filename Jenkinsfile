pipeline {
    agent {
        label 'AGENT-1'
    }

    environment {
        APP_VERSION = ''
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
            description: 'Toggle this value'
        )
    }

    stages {

        stage('Read package.json') {
            steps {
                script {
                    sh '''
                        echo "===== package.json ====="
                        cat package.json
                        echo "========================"
                    '''

                    def packageJson = readJSON file: 'package.json'
                    def appVersion = packageJson.version

                    echo "Package name: ${packageJson.name}"
                    echo "Package version: ${appVersion}"

                    if (!appVersion) {
                        error "❌ package.json version is missing"
                    }

                    env.APP_VERSION = appVersion.toString()

                    echo "APP_VERSION: ${env.APP_VERSION}"
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    npm install
                '''
            }
        }

        stage('Unit Testing') {
            steps {
                sh '''
                    echo "unit tests"
                '''
            }
        }

        /*
        stage('Sonar Scan') {
            environment {
                scannerHome = tool 'sonar-8.1'
            }
            steps {
                script {
                    withSonarQubeEnv(installationName: 'sonar-8.1') {
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
                        def severity = alert?.security_advisory?.severity?.toLowerCase()
                        def state = alert?.state?.toLowerCase()

                        return state == 'open' &&
                               (severity == 'critical' || severity == 'high')
                    }

                    if (criticalOrHigh.size() > 0) {
                        error "❌ Found ${criticalOrHigh.size()} HIGH/CRITICAL Dependabot alerts. Failing pipeline!"
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
                        region: 'us-east-1'
                    ) {

                        sh """
                            echo "Building Docker image with version: ${env.APP_VERSION}"

                            aws ecr get-login-password \
                              --region ${REGION} | \
                            docker login \
                              --username AWS \
                              --password-stdin \
                              ${ACC_ID}.dkr.ecr.us-east-1.amazonaws.com

                            docker build \
                              -t ${ACC_ID}.dkr.ecr.us-east-1.amazonaws.com/${PROJECT}/${COMPONENT}:${env.APP_VERSION} .

                            docker push \
                              ${ACC_ID}.dkr.ecr.us-east-1.amazonaws.com/${PROJECT}/${COMPONENT}:${env.APP_VERSION}
                        """
                    }
                }
            }
        }

        stage('Check Scan Results') {
            steps {
                script {
                    withAWS(
                        credentials: 'aws-creds',
                        region: 'us-east-1'
                    ) {

                        def findings = null
                        def scanComplete = false

                        for (int i = 1; i <= 20; i++) {

                            echo "Checking ECR scan... Attempt ${i}/20"

                            def result = sh(
                                script: """
                                    aws ecr describe-image-scan-findings \
                                      --repository-name ${PROJECT}/${COMPONENT} \
                                      --image-id imageTag=${env.APP_VERSION} \
                                      --region ${REGION} \
                                      --output json
                                """,
                                returnStatus: true
                            )

                            if (result == 0) {

                                findings = sh(
                                    script: """
                                        aws ecr describe-image-scan-findings \
                                          --repository-name ${PROJECT}/${COMPONENT} \
                                          --image-id imageTag=${env.APP_VERSION} \
                                          --region ${REGION} \
                                          --output json
                                    """,
                                    returnStdout: true
                                ).trim()

                                def scanJson = readJSON text: findings

                                def status = scanJson.imageScanStatus?.status

                                echo "ECR scan status: ${status}"

                                if (status == 'COMPLETE') {
                                    scanComplete = true
                                    break
                                }

                            } else {
                                echo "⏳ ECR scan result not available yet..."
                            }

                            sleep 15
                        }

                        if (!scanComplete) {
                            error "❌ ECR image scan did not complete within the expected time."
                        }

                        def json = readJSON text: findings

                        def highCritical = json.imageScanFindings.findAll {
                            it.severity == 'HIGH' ||
                            it.severity == 'CRITICAL'
                        }

                        if (highCritical.size() > 0) {

                            echo "❌ Found ${highCritical.size()} HIGH/CRITICAL vulnerabilities!"

                            error("Build failed due to vulnerabilities")

                        } else {

                            echo "✅ No HIGH/CRITICAL vulnerabilities found."
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

                    echo "Triggering catalogue-cd with version ${env.APP_VERSION}"

                    build job: 'catalogue-cd',
                    parameters: [
                        string(
                            name: 'appVersion',
                            value: "${env.APP_VERSION}"
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