pipeline {
    agent any

    environment {
        BRIDGE_CLI_DIR = "${WORKSPACE}/bridge-cli"
        DETECT_PROJECT_NAME = "qe-ninja-blackduck-project-qa"
        DETECT_VERSION_NAME = "1.0.0"
        GO_VERSION="1.21.2"
        BD_URL = credentials('BLACKDUCK_URL') // or use credentials if you really want
        BD_TOKEN = credentials('BLACKDUCK_API_TOKEN') // must be 'Secret text'
    }
    
    triggers {
        cron '15 03 * * 1-5' // Runs at 03:15 on every day-of-week from Monday through Friday
         }
    
    stages {
        stage('Download and Extract Bridge CLI') {
            steps {
                sh '''
                    mkdir -p "$BRIDGE_CLI_DIR"

                    # Download with error check
                    curl -f -L "https://repo.blackduck.com/bds-integrations-release/com/blackduck/integration/bridge/binaries/bridge-cli-bundle/latest/bridge-cli-bundle-linux64.zip" \
                        -o bridge.zip

                    # Extract with jar (no unzip needed)
                    (cd "$BRIDGE_CLI_DIR" && jar -xf ../bridge.zip)

                    # check if the binary exists
                    ls -lrt ${BRIDGE_CLI_DIR}
                    chmod +x "$BRIDGE_CLI_DIR"/bridge-cli-bundle-linux64/bridge-cli

                    # Verify
                    "$BRIDGE_CLI_DIR/bridge-cli-bundle-linux64/bridge-cli" --version
                '''
            }
        }

        stage('Prepare Bridge CLI') {
            steps {
                sh '''
                    chmod -R +x bridge-cli/bridge-cli-bundle-linux64/adapters
                '''
            }
        }


        stage('Run Black Duck Bridge CLI with SARIF Output') {
            steps {
                sh """
                    
                    curl -LO https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz
                    rm -rf /tmp/go
                    tar -C /tmp -xzf go${GO_VERSION}.linux-amd64.tar.gz
                    export PATH=/tmp/go/bin:$PATH

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

        // stage('Archive SARIF Report') {
        //     steps {
        //         archiveArtifacts artifacts: 'output/*.sarif', fingerprint: true
        //     }
        // }
    }
}
