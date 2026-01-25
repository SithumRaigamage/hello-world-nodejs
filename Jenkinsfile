pipeline {
    agent any

    // Define global environment variables including SonarQube details
    environment {
        // SonarQube Configuration
        // 'SonarScanner' must be configured in Jenkins Global Tool Configuration
        SCANNER_HOME = tool 'SonarScanner'
        
        // SonarQube Server Details
        // NOTE: In a production environment, use Credentials Binding for tokens!
        SONAR_SERVER_URL = 'http://host.docker.internal:9000'
        SONAR_AUTH_TOKEN = credentials('sonarqube-token') 
        SONAR_PROJECT_KEY = 'hello-world-nodejs'
        SONAR_PROJECT_NAME = 'Hello World Node.js'
        
        // Trivy Reporting
        TRIVY_REPORT_NAME = 'trivy-report.html'
        
        // Sender and Receiver Emails for Notifications
        SENDER_EMAIL = 'sunilsithum@gmail.com' 
        RECEIVER_EMAIL = 'sraig2002@gmail.com' 
    }

    tools {
        nodejs 'NODEJS'
    }

    parameters {
        string(name: 'RELEASE_VERSION', defaultValue: '', description: 'Semantic Release Version (e.g. 1.0.0)')
        string(name: 'GIT_REPO_URL', defaultValue: 'https://github.com/SithumRaigamage/hello-world-nodejs.git', description: 'Git Repository URL')
        string(name: 'BRANCH', defaultValue: 'main', description: 'Branch to build')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'qa', 'staging', 'prod'], description: 'Target Environment')
        booleanParam(name: 'SEND_EMAIL', defaultValue: true, description: 'Send email notifications?')
    }

    stages {
        // Stage to checkout source code from the provided Git repository and branch
        stage('Checkout') {
            steps {
                script {
                    // Disable implicit checkout if desired using 'options { skipDefaultCheckout() }' at top level
                    // Explicit checkout:
                    checkout([$class: 'GitSCM', 
                        branches: [[name: "*/${params.BRANCH}"]], 
                        userRemoteConfigs: [[url: "${params.GIT_REPO_URL}"]]
                    ])
                }
            }
        }

        // Stage to verify that the provided release version matches semantic versioning (X.Y.Z)
        stage('Verify Version') {
            steps {
                script {
                    def versionPattern = /^\d+\.\d+\.\d+$/ 
                    if (params.RELEASE_VERSION =~ versionPattern) {
                        echo "Version ${params.RELEASE_VERSION} is valid."
                    } else {
                        error "Version ${params.RELEASE_VERSION} is NOT valid SemVer (X.Y.Z)."
                    }
                }
            }
        }
        
        // Stage to install Node.js dependencies
        stage('Install') {
            steps {
                sh 'npm install'
            }
        }

        // Stage to perform static code analysis using SonarQube
        stage('SonarQube Analysis') {
            steps {
                // 'SonarServer' is the name of the SonarQube server configured in Jenkins
                withSonarQubeEnv('SonarServer') { 
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.projectName="${SONAR_PROJECT_NAME}" \
                        -Dsonar.host.url=${SONAR_SERVER_URL} \
                        -Dsonar.login=${SONAR_AUTH_TOKEN} \
                        -Dsonar.sources=. \
                        -Dsonar.exclusions=node_modules/**
                    '''
                }
            }
        }

        // Stage to logicially determine the Docker image name based on the target environment
        stage('Determine Image Name') {
            steps {
                script {
                    def imageName = "hello-world-nodejs"
                    // Prefix non-production builds with the environment name
                    if (params.ENVIRONMENT != 'prod') {
                        imageName = "${params.ENVIRONMENT}-${imageName}"
                    }
                    env.FINAL_IMAGE_NAME = imageName
                    echo "Determined Environment Image Name: ${env.FINAL_IMAGE_NAME}"
                }
            }
        }

        // Stage to build the Docker image with the determined name and version tag
        stage('Build Image') {
            steps {
                script {
                    sh "docker build -t ${env.FINAL_IMAGE_NAME}:${params.RELEASE_VERSION} ."
                }
            }
        }

        // Stage to scan the built Docker image for vulnerabilities using Trivy
        stage('Trivy Security Scan') {
            steps {
                script {
                    try {
                        // Download the official HTML template dynamically
                        sh "wget -q https://github.com/aquasecurity/trivy/raw/main/contrib/html.tpl -O html.tpl"
                        // Run Trivy scan using the downloaded template
                        sh "trivy image --format template --template @html.tpl -o ${TRIVY_REPORT_NAME} ${env.FINAL_IMAGE_NAME}:${params.RELEASE_VERSION}"
                    } catch (Exception e) {
                        echo "WARNING: Trivy scan failed: ${e.message}. Continuing pipeline..."
                    }
                }
            }
        }
    }

    // Post-build conditions to handle reporting, status notifications, and emails
    post {
        always {
            script {
                // Only archive and publish if the report was actually generated
                if (fileExists(env.TRIVY_REPORT_NAME)) {
                    archiveArtifacts artifacts: "${env.TRIVY_REPORT_NAME}", allowEmptyArchive: true
                    
                    publishHTML (target: [
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: "${env.TRIVY_REPORT_NAME}",
                        reportName: 'Trivy Security Report'
                    ])
                } else {
                    echo "SKIPPING ARTIFACTS: ${env.TRIVY_REPORT_NAME} was not generated."
                }
            }
        }
        success {
            script {
                if (params.SEND_EMAIL) {
                    try {
                        emailext (
                            from: "${env.SENDER_EMAIL}",
                            to: "${env.RECEIVER_EMAIL}",
                            subject: "BUILD SUCCESS: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                            body: generateEmailBody('SUCCESS', '#28a745'),
                            mimeType: 'text/html'
                        )
                    } catch (Exception e) {
                        echo "ERROR: Failed to send success email: ${e.message}"
                    }
                }
            }
            echo 'Pipeline completed successfully. Email notification sent.'
        }
        failure {
            script {
                if (params.SEND_EMAIL) {
                    try {
                        emailext (
                            from: "${env.SENDER_EMAIL}",
                            to: "${env.RECEIVER_EMAIL}",
                            subject: "BUILD FAILED: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                            body: generateEmailBody('FAILURE', '#dc3545'),
                            mimeType: 'text/html'
                        )
                    } catch (Exception e) {
                        echo "ERROR: Failed to send failure email: ${e.message}"
                    }
                }
            }
            echo 'Pipeline failed. Email notification sent.'
        }
    }
}

// Helper function to generate an HTML email body
def generateEmailBody(status, color) {
    return """
    <html>
    <head>
        <style>
            body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background-color: #f4f4f4; padding: 20px; }
            .container { max-width: 600px; margin: 0 auto; background-color: #ffffff; border-radius: 8px; overflow: hidden; box-shadow: 0 4px 6px rgba(0,0,0,0.1); }
            .header { background-color: ${color}; color: #ffffff; padding: 20px; text-align: center; }
            .header h1 { margin: 0; font-size: 24px; }
            .content { padding: 30px; }
            .info-item { margin-bottom: 15px; border-bottom: 1px solid #eeeeee; padding-bottom: 10px; }
            .info-item:last-child { border-bottom: none; }
            .label { font-weight: bold; color: #555555; display: inline-block; width: 120px; }
            .value { color: #333333; }
            .footer { background-color: #eeeeee; padding: 15px; text-align: center; font-size: 12px; color: #777777; }
            .button { display: inline-block; padding: 10px 20px; background-color: ${color}; color: #ffffff; text-decoration: none; border-radius: 4px; margin-top: 20px; }
        </style>
    </head>
    <body>
        <div class="container">
            <div class="header">
                <h1>Build ${status}</h1>
            </div>
            <div class="content">
                <div class="info-item">
                    <span class="label">Project:</span>
                    <span class="value">${env.JOB_NAME}</span>
                </div>
                <div class="info-item">
                    <span class="label">Build Number:</span>
                    <span class="value">#${env.BUILD_NUMBER}</span>
                </div>
                <div class="info-item">
                    <span class="label">Branch:</span>
                    <span class="value">${params.BRANCH}</span>
                </div>
                <div class="info-item">
                    <span class="label">Environment:</span>
                    <span class="value">${params.ENVIRONMENT}</span>
                </div>
                <div class="info-item">
                    <span class="label">Version:</span>
                    <span class="value">${params.RELEASE_VERSION}</span>
                </div>
                <div style="text-align: center;">
                    <a href="${env.BUILD_URL}" class="button">View Build Logs</a>
                </div>
            </div>
            <div class="footer">
                <p>Generated by Jenkins CI</p>
            </div>
        </div>
    </body>
    </html>
    """
}
