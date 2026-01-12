pipeline {
    agent any

    parameters {
        string(name: 'RELEASE_VERSION', defaultValue: '', description: 'Semantic Release Version (e.g. 1.0.0)')
        string(name: 'BRANCH', defaultValue: 'main', description: 'Branch to build')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'qa', 'staging', 'prod'], description: 'Target Environment')
    }

    stages {
        stage('Verify Version') {
            steps {
                script {
                    // Start of Anchor, Start of decimal digits, dot, decimal digits, dot, decimal digits, end of anchor
                    def versionPattern = /^\d+\.\d+\.\d+$/ 
                    if (params.RELEASE_VERSION =~ versionPattern) {
                        echo "Version ${params.RELEASE_VERSION} is valid."
                    } else {
                        error "Version ${params.RELEASE_VERSION} is NOT valid SemVer (X.Y.Z)."
                    }
                }
            }
        }
        
        stage('Install') {
            steps {
                sh 'npm install'
            }
        }

        stage('Test') {
            steps {
                sh 'npm test'
            }
        }

        stage('Determine Image Name') {
            steps {
                script {
                    def imageName = "hello-world-nodejs"
                    if (params.ENVIRONMENT != 'prod') {
                        imageName = "${params.ENVIRONMENT}-${imageName}"
                    }
                    env.FINAL_IMAGE_NAME = imageName
                    echo "Determined Environment Image Name: ${env.FINAL_IMAGE_NAME}"
                }
            }
        }

        stage('Build Image') {
            steps {
                script {
                    // Using the determined name and the release version as tag
                    sh "docker build -t ${env.FINAL_IMAGE_NAME}:${params.RELEASE_VERSION} ."
                }
            }
        }
    }
}
