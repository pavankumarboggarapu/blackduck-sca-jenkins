pipeline {
    agent any

    environment {
        GITHUB_TOKEN = credentials('github-pkumarcoverity')
        DETECT_EXCLUDED_DETECTOR_TYPES=GIT
    }

    tools {
        maven 'maven-3'
        jdk 'openjdk-21'
    }

    stages {
        stage('Init') {
            steps {
                script {
                    // Get repo name from Git URL
                    def gitUrl = sh(script: "git config --get remote.origin.url", returnStdout: true).trim()
                    env.REPO_NAME = gitUrl.tokenize('/.git')[-1]

                    // Get current branch name
                    env.BRANCH_NAME = sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim()

                    echo "Repo Name: ${env.REPO_NAME}"
                    echo "Branch Name: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn -B package'
            }
        }

        stage('Coverity Scan') {
            steps {
                security_scan product: 'blackducksca',
                    blackducksca_scan_failure_severities: 'BLOCKER',
                    blackducksca_prComment_enabled: true,
                    blackducksca_reports_sarif_create: true,
                    mark_build_status: 'UNSTABLE',
                    github_token: "${env.GITHUB_TOKEN}",
                    include_diagnostics: false
            }
        }
    }

    post {
        always {
            archiveArtifacts allowEmptyArchive: true, artifacts: '.bridge/bridge.log, .bridge/*/idir/build-log.txt'
            cleanWs()
        }
    }
}
