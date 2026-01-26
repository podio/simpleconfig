
pipeline {
    agent {
        node {
            label 'azure-linux-ubuntu-18'
        }
    }
    options {
        ansiColor('xterm')
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('TruffleHog Scan') {
            steps {
                script {
                    echo "Running TruffleHog scan..."
                    sh '''
                        docker run --rm -v ${WORKSPACE}:/usr/src \
                            artifacts.progress.com/ci-local-docker/trufflesecurity/trufflehog:3.88.29-amd64 \
                            git file:/usr/src \
                            --results=verified,unknown \
                            --force-skip-binaries \
                            --force-skip-archives \
                            --json \
                            --no-update \
                            > trufflehog_report.json
                    '''
                    echo "📄 TruffleHog scan completed. Output saved to trufflehog_report.json"
                    def secretsFound = sh(
                        script: 'grep -q \'"SourceMetadata"\' trufflehog_report.json',
                        returnStatus: true
                    )
                    if (secretsFound == 0) {
                        echo "❌ TruffleHog found potential secrets!"
                        sh 'cat trufflehog_report.json'
                        error("❌ TruffleHog scan failed, potential secrets detected in the repository.")
                    } else {
                        echo "✅ TruffleHog scan passed - No secrets detected."
                    }
                }
            }
        }
        stage('polaris') {
            steps {
                withCredentials([string(credentialsId: 'blackduck-api-token', variable: 'BRIDGE_POLARIS_ACCESSTOKEN')]) {
                    script {
                        try {
                            def status = sh(
                                returnStatus: true,
                                script: """
                                    bridge-cli --stage polaris \
                                    polaris.serverurl="https://polaris.blackduck.com" \
                                    polaris.application.name="Podio-Podio" \
                                    polaris.project.name="simpleconfig" \
                                    polaris.branch.name="${env.BRANCH_NAME}" \
                                    polaris.assessment.types="SAST" \
                                    polaris.assessment.mode="SOURCE_UPLOAD"
                                """
                            )
                            if (status == 8) {
                                unstable('Policy violation')
                            } else if (status != 0) {
                                error('Bridge CLI failure')
                            }
                        } catch (Exception e) {
                            echo "${e.getMessage()}"
                        }
                    }
                }
            }
        }
        stage('blackducksca') {
            steps {
                script {
                    try {
                        withCredentials([string(credentialsId: 'blackduck-sca-token', variable: 'BRIDGE_BLACKDUCKSCA_TOKEN')]) {
                            withEnv([
                                'BRIDGE_BLACKDUCKSCA_URL=https://progresssoftware.app.blackduck.com',
                                'BRIDGE_BLACKDUCKSCA_SCAN_FAILURE_SEVERITIES=CRITICAL',
                                'BRIDGE_BLACKDUCKSCA_SCAN_FULL=true',
                                "BRIDGE_DETECT_ARGS=--detect.project.name=DX-Podio-simpleconfig --detect.project.version.name=${env.BRANCH_NAME} --detect.project.version.update=true --detect.project.group.name=Podio-Podio"
                            ]) {
                                sh(returnStatus: true, script: 'bridge-cli --stage blackducksca')
                            }
                        }
                    } catch (Exception e) {
                        echo "${e.getMessage()}"
                    }
                }
            }
        }
    }
}    
