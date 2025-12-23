pipeline {
    agent any

    environment {
        TF_IN_AUTOMATION = '1'
        TF_CLI_ARGS = '-no-color'
    }

    stages {

        // =========================
        // STAGE 1: TERRAFORM INIT
        // =========================
        stage('Init') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY

                        echo "Initializing Terraform..."
                        terraform init
                    '''
                }
            }
        }

        // ======================================
        // STAGE 2: INSPECT BRANCH TFVARS (TASK 3)
        // ======================================
        stage('Inspect TFVARS') {
            steps {
                sh '''
                    echo "Inspecting tfvars for branch: $BRANCH_NAME"

                    if [ -f ${BRANCH_NAME}.tfvars ]; then
                        echo "===== ${BRANCH_NAME}.tfvars ====="
                        cat ${BRANCH_NAME}.tfvars
                        echo "==============================="
                    else
                        echo "ERROR: ${BRANCH_NAME}.tfvars not found"
                        exit 1
                    fi
                '''
            }
        }

        // ======================================
        // STAGE 3: TERRAFORM PLAN (TASK 4)
        // ======================================
        stage('Plan') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY

                        echo "Running Terraform plan for branch: $BRANCH_NAME"

                        terraform plan \
                          -var-file=${BRANCH_NAME}.tfvars \
                          -out=plan-${BRANCH_NAME}.tfplan

                        terraform show -no-color plan-${BRANCH_NAME}.tfplan
                    '''
                }
            }
        }

        // =================================================
        // STAGE 4: MANUAL APPROVAL (DEV ONLY) – TASK 5
        // =================================================
        stage('Validate & Apply') {
            when {
                branch 'dev'
            }
            steps {
                input message: 'Approve apply to dev?', ok: 'Apply'

                sh '''
                    echo "Apply approved for branch: $BRANCH_NAME"
                    echo "Terraform apply intentionally skipped"
                    echo "Reason: BYOD-3 focuses on infrastructure planning & approval workflow"
                '''
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully for branch: ${env.BRANCH_NAME}"
        }
        failure {
            echo "Pipeline failed for branch: ${env.BRANCH_NAME}"
        }
    }
}
