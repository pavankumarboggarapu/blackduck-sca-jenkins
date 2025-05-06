// example Jenkinsfile for standalone Black Duck, Coverity and SRM (Code DX) using Jenkins Plugins
// - Black Duck and Coverity full scans on push to specified branches with import into SRM
// - Black Duck Rapid and Coverity Comparison scans on pull requests with PR comments enabled
// https://plugins.jenkins.io/blackduck-security-scan/
// https://plugins.jenkins.io/codedx
pipeline {
    agent { label 'linux64' }
    environment {
        REPO_NAME = "${env.GIT_URL.tokenize('/.')[-2]}"
        FULLSCAN = "${env.BRANCH_NAME ==~ /^(main|master|develop|stage|release)$/ ? 'true' : 'false'}"
        PRSCAN = "${env.CHANGE_TARGET ==~ /^(main|master|develop|stage|release)$/ ? 'true' : 'false'}"
        GITHUB_TOKEN = credentials('github-pat')
        DETECT_PROJECT_NAME = "${env.REPO_NAME}"
    }
    tools {
        maven 'maven-3'
        jdk 'openjdk-21'
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn -B compile'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn -B test'
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
                    coverity_local: true,
                    coverity_install_directory: "$COVERITY_TOOL_HOME",
                    mark_build_status: 'UNSTABLE',
                    github_token: "$GITHUB_TOKEN",
                    include_diagnostics: false
            }
        }
        stage('Black Duck') {
            when {
                anyOf {
                    environment name: 'FULLSCAN', value: 'true'
                    environment name: 'PRSCAN', value: 'true'
                }
            }
            steps {
                security_scan product: 'blackducksca',
                    blackducksca_scan_failure_severities: 'BLOCKER',
                    blackducksca_prComment_enabled: true,
                    blackducksca_reports_sarif_create: true,
                    mark_build_status: 'UNSTABLE',
                    github_token: "$GITHUB_TOKEN",
                    include_diagnostics: false
            }
        }
        stage('SRM') {
            when { environment name: 'FULLSCAN', value: 'true' }
            steps {
                step([
                    $class: 'CodeDxPublisher',
                    url: "$SRM_URL",
                    selfSignedCertificateFingerprint: '',
                    keyCredentialId: 'srm.field-test.blackduck.com',
                    project: [$class: 'NamedProject', projectName: "chuckaude-$REPO_NAME", autoCreate: true],
                    baseBranchName: 'main',
                    targetBranchName: "$BRANCH_NAME",
                    // no option to only use analysis import, see ER CDX-1700
                    sourceAndBinaryFiles: '**',
                    // include then exclude everything workaround no longer works
                    //excludedSourceAndBinaryFiles: '**',
                    analysisName: "Build [#$BUILD_TAG]($BUILD_URL)",
                    analysisResultConfiguration: [
                        policyBreakBuildBehavior: 'MarkUnstable',
                        failureSeverity: 'High',
                        failureOnlyNew: true,
                        unstableSeverity: 'High',
                        unstableOnlyNew: false,
                        numBuildsInGraph: 0
                    ]
                ])
            }
        }
        stage('Deploy') {
            steps {
                sh 'mvn -B install'
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
