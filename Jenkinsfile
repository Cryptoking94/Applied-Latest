pipeline {
    agent any

    parameters {
        string(name: 'TFVARS_FILE_PATH', defaultValue: 'terraform.tfvars', description: 'Path to the .tfvars file')
        string(name: 'JIRA_ISSUE_KEY', defaultValue: 'TER-6', description: 'Jira Issue Key')
    }

    environment {
        AWS_CREDENTIALS_ID = 'aws'
        INFRACOST_API_KEY = credentials('infra-cost-api')
        JIRA_EMAIL = credentials('username')
        JIRA_API_TOKEN = credentials('latest-api')
        COST_THRESHOLD = 50.00
        JIRA_API_URL = 'https://hashirmujeeb94.atlassian.net/rest/api/2/issue'
        CUSTOM_FIELD_ID = '10039'
        POLL_INTERVAL = 60
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    env.FAILED_STAGE = 'Checkout'
                }
                git url: 'https://github.com/Cryptoking94/code_infra', branch: 'main'
            }
        }

        stage('Terraform Init') {
            steps {
                script {
                    env.FAILED_STAGE = 'Terraform Init'
                }
                withCredentials([aws(credentialsId: "${AWS_CREDENTIALS_ID}")]) {
                    sh 'terraform init'
                }
            }
        }

        stage('Terraform Validate') {
            steps {
                script {
                    env.FAILED_STAGE = 'Terraform Validate'
                }
                sh 'terraform validate'
            }
        }
/*
        stage('TFLint') {
            steps {
                script {
                    env.FAILED_STAGE = 'TFLint'
                }
                sh 'tflint'
            }
        }

        stage('TFsec') {
            steps {
                script {
                    env.FAILED_STAGE = 'TFsec'
                }
                sh 'tfsec .'
            }
        }

        stage('Checkov') {
            steps {
                script {
                    env.FAILED_STAGE = 'Checkov'
                }
                sh 'checkov -d .'
            }
        }
*/
        stage('Terraform Plan') {
            steps {
                script {
                    env.FAILED_STAGE = 'Terraform Plan'
                }
                withCredentials([aws(credentialsId: "${AWS_CREDENTIALS_ID}")]) {
                    sh "terraform plan -out=tfplan -var-file=${params.TFVARS_FILE_PATH}"
                }
            }
        }

        stage('Infracost Cost Estimation') {
            steps {
                script {
                    env.FAILED_STAGE = 'Infracost Cost Estimation'
                }
                withCredentials([string(credentialsId: 'infra-cost-api', variable: 'INFRACOST_API_KEY')]) {
                    sh 'infracost breakdown --path=tfplan --format=json --out-file=infracost.json'
                    sh 'infracost output --path=infracost.json --format=table'
                }
            }
        }

        stage('Check Cost Threshold') {
            steps {
                script {
                    env.FAILED_STAGE = 'Check Cost Threshold'
                    try {
                        def costString = sh(script: "jq -r '.totalMonthlyCost' infracost.json", returnStdout: true).trim()
                        if (costString == 'null' || !costString) {
                            error "Failed to retrieve cost from infracost.json. The cost value is null or missing."
                        }
                        def cost = costString.toDouble()
                        def jiraIssueKey = params.JIRA_ISSUE_KEY
                        def credentials = "${JIRA_EMAIL}:${JIRA_API_TOKEN}"
                        def encodedCredentials = credentials.bytes.encodeBase64().toString()

                        if (cost > env.COST_THRESHOLD.toDouble()) {
                            // Add comment to Jira issue indicating the cost exceeds threshold
                            def errorMsg = "Estimated cost of \$${cost} exceeds the threshold of \$${env.COST_THRESHOLD}. Awaiting approval."

                            sh """
                                curl -v --http1.1 -X POST -H "Content-Type: application/json" \
                                     -H "Authorization: Basic ${encodedCredentials}" \
                                     -d '{
                                           "body": "${errorMsg}"
                                         }' \
                                     ${JIRA_API_URL}/${jiraIssueKey}/comment
                            """

                            // Wait for approval from the Jira custom field
                            def authHeader = 'Basic ' + "${JIRA_EMAIL}:${JIRA_API_TOKEN}".bytes.encodeBase64().toString()
                            def fieldValue = 'none'

                            while (fieldValue == 'none' || fieldValue == null) {
                                echo "Checking approval status..."

                                def response = sh(
                                    script: """
                                    curl -s -H 'Authorization: ${authHeader}' \
                                    -H 'Accept: application/json' \
                                    ${JIRA_API_URL}/${jiraIssueKey}?fields=customfield_${CUSTOM_FIELD_ID}
                                    """,
                                    returnStdout: true
                                ).trim()

                                echo "API Response: ${response}"

                                def jsonResponse = readJSON(text: response)
                                def customField = jsonResponse.fields["customfield_${CUSTOM_FIELD_ID}"]

                                if (customField != null && customField != net.sf.json.JSONNull.getInstance()) {
                                    fieldValue = customField.value
                                } else {
                                    fieldValue = 'none'
                                }

                                if (fieldValue == 'Approve') {
                                    echo "The field status is: APPROVE"
                                    // Add a comment that the manager approved the cost
                                    def approvalMsg = "Manager approved the cost and Infra is deploying."
                                    sh """
                                        curl -v --http1.1 -X POST -H "Content-Type: application/json" \
                                             -H "Authorization: Basic ${encodedCredentials}" \
                                             -d '{
                                                   "body": "${approvalMsg}"
                                                 }' \
                                             ${JIRA_API_URL}/${jiraIssueKey}/comment
                                    """
                                    break
                                } else if (fieldValue == 'Discard') {
                                    echo "The field status is: Discard"
                                    // Add a comment that the manager declined the cost
                                    def declineMsg = "Manager Declined the cost, You need to adjust the sizing."
                                    sh """
                                        curl -v --http1.1 -X POST -H "Content-Type: application/json" \
                                             -H "Authorization: Basic ${encodedCredentials}" \
                                             -d '{
                                                   "body": "${declineMsg}"
                                                 }' \
                                             ${JIRA_API_URL}/${jiraIssueKey}/comment
                                    """
                                    error "Pipeline declined by manager."
                                } else {
                                    echo "Status is still 'none'. Waiting for approval or decline..."
                                    sleep(POLL_INTERVAL)
                                }
                            }
                        } else {
                            // Add comment to Jira issue that cost is within the limit
                            def withinLimitMsg = "Cost is within the limit, Infra is Deploying."
                            sh """
                                curl -v --http1.1 -X POST -H "Content-Type: application/json" \
                                     -H "Authorization: Basic ${encodedCredentials}" \
                                     -d '{
                                           "body": "${withinLimitMsg}"
                                         }' \
                                     ${JIRA_API_URL}/${jiraIssueKey}/comment
                            """
                        }
                    } catch (Exception e) {
                        error "Stage failed: ${env.FAILED_STAGE}. Reason: ${e.message}"
                    }
                }
            }
        }

        stage('Final Approval Success') {
            when {
                expression {
                    return env.FIELD_VALUE == 'Approve'
                }
            }
            steps {
                echo "Pipeline completed successfully: APPROVED!"
            }
        }
    }

    post {
        failure {
            script {
                def failedStage = env.FAILED_STAGE ?: "Unknown Stage"
                def errorMsg = "Jenkins job failed in stage ${failedStage}."
                def credentials = "${JIRA_EMAIL}:${JIRA_API_TOKEN}"
                def encodedCredentials = credentials.bytes.encodeBase64().toString()
                def jiraIssueKey = params.JIRA_ISSUE_KEY

                sh """
                    curl -v --http1.1 -X POST -H "Content-Type: application/json" \
                         -H "Authorization: Basic ${encodedCredentials}" \
                         -d '{
                               "body": "${errorMsg}"
                             }' \
                         ${JIRA_API_URL}/${jiraIssueKey}/comment
                """
            }
        }
        always {
            cleanWs()
        }
    }
}
