// example Jenkinsfile for Coverity scans using the Black Duck Security Scan Plugin
// https://plugins.jenkins.io/blackduck-security-scan
pipeline {
    agent { label 'linux64' }
    environment {
        REPO_NAME = "${env.GIT_URL.tokenize('/.')[-2]}"
        FULLSCAN = "${env.BRANCH_NAME ==~ /^(main|master|develop|stage|release)$/ ? 'true' : 'false'}"
        PRSCAN = "${env.CHANGE_TARGET ==~ /^(main|master|develop|stage|release)$/ ? 'true' : 'false'}"
        GITHUB_TOKEN = credentials('github-pat')
    }
    tools {
        maven 'maven-3'
        jdk 'openjdk-21'
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn -B package'
            }
        }
        stage('Coverity') {
            when {
                anyOf {
                    environment name: 'FULLSCAN', value: 'true'
                    environment name: 'PRSCAN', value: 'true'
                }
            }
            steps {
                security_scan product: 'coverity',
                    coverity_project_name: "$REPO_NAME",
                    coverity_stream_name: "$REPO_NAME-$BRANCH_NAME",
                    coverity_args: "-o commit.connect.description=$BUILD_TAG",
                    coverity_policy_view: 'Outstanding Issues',
                    coverity_prComment_enabled: true,
                    mark_build_status: 'UNSTABLE',
                    github_token: "$GITHUB_TOKEN",
                    include_diagnostics: false
            }
        }
    }
    post {
        always {
            archiveArtifacts allowEmptyArchive: true, artifacts: '.bridge/bridge.log, .bridge/*/idir/build-log.txt'
            //zip archive: true, dir: '.bridge', zipFile: 'bridge-logs.zip'
            cleanWs()
        }
    }
}
