pipeline {
    agent any

    environment {
        BRIDGE_CLI_DIR = "${WORKSPACE}/bridge-cli"
        DETECT_PROJECT_NAME = "qe-ninja-blackduck-project-qa"
        DETECT_VERSION_NAME = "1.0.0"
        GO_VERSION="1.21.2"
        BD_URL = credentials('BLACKDUCK_URL')
        BD_TOKEN = credentials('BLACKDUCK_API_TOKEN')
    }

    triggers {
        cron '15 23 * * 1,4' // Runs at 23:15 on Monday and Thursday
    }

    stages {
        stage('Run Black Duck Bridge CLI with SARIF Output') {
            steps {
                sh """
                    # Install required dependencies
                    apt-get update && apt-get install -y curl tar default-jdk || {
                        echo "apt-get not available, trying yum..."
                        yum install -y curl tar java-11-openjdk || {
                            echo "Warning: Could not install dependencies. Proceeding anyway..."
                        }
                    }

                    # Clean up any previous Bridge CLI installation
                    rm -rf "${BRIDGE_CLI_DIR}"
                    mkdir -p "${BRIDGE_CLI_DIR}"

                    # Download and extract Bridge CLI
                    curl -f -L "https://repo.blackduck.com/bds-integrations-release/com/blackduck/integration/bridge/binaries/bridge-cli-bundle/latest/bridge-cli-bundle-linux64.zip" \
                        -o bridge.zip
                    jar -xf bridge.zip
                    mv bridge-cli-bundle-linux64 "${BRIDGE_CLI_DIR}/"
                    chmod -R +x "${BRIDGE_CLI_DIR}/bridge-cli-bundle-linux64"

                    # Verify Bridge CLI
                    "${BRIDGE_CLI_DIR}/bridge-cli-bundle-linux64/bridge-cli" --version

                    # Download and install Go
                    curl -LO https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz
                    rm -rf /tmp/go
                    tar -C /tmp -xzf go${GO_VERSION}.linux-amd64.tar.gz
                    export PATH=/tmp/go/bin:\$PATH

                    # Verify Go installation
                    /tmp/go/bin/go version

                    # Create output directory for SARIF report
                    mkdir -p output

                    # Run Bridge CLI scan
                    "${BRIDGE_CLI_DIR}/bridge-cli-bundle-linux64/bridge-cli" \
                        --stage blackducksca \
                        blackducksca.url="https://blackduck.saas-preprod.beescloud.com/" \
                        blackducksca.scan.full=true \
                        blackducksca.token="${BD_TOKEN}" \
                        blackducksca_reports_sarif_create=true \
                        blackducksca_reports_sarif_file_path="output/blackduck-sarif-report.sarif" \
                        blackducksca_reports_sarif_groupSCAIssues=false
                """
            }
        }

        stage('Check the SARIF Report') {
            steps {
                sh '''
                    echo "Checking SARIF report..."
                    ls -l output/*.sarif
                    cat output/*.sarif
                '''
            }
        }

        stage('Security Scan') {
            steps {
                registerSecurityScan(
                    // Security Scan to include
                    artifacts: "output/blackduck-sarif-report.sarif",
                    format: "sarif",
                    archive: true
                )
            }
        }
    }
}
