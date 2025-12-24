pipeline {
    agent any

    environment {
        TF_IN_AUTOMATION = 'true'
        TF_INPUT = 'false'
        ANSIBLE_HOST_KEY_CHECKING = 'False'
        AWS_REGION = 'us-east-1'
    }

    stages {

        // =====================================================
        // STAGE 1: CHECKOUT
        // =====================================================
        stage('📥 Checkout') {
            steps {
                checkout scm
            }
        }

        // =====================================================
        // STAGE 2: TERRAFORM INIT
        // =====================================================
        stage('🔧 Terraform Init') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                      export AWS_ACCESS_KEY_ID
                      export AWS_SECRET_ACCESS_KEY
                      terraform init
                    '''
                }
            }
        }

        // =====================================================
        // STAGE 3: TERRAFORM APPLY
        // =====================================================
        stage('🚀 Terraform Apply') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                      export AWS_ACCESS_KEY_ID
                      export AWS_SECRET_ACCESS_KEY

                      terraform apply \
                        -auto-approve \
                        -var-file=${BRANCH_NAME}.tfvars
                    '''
                }
            }
        }

        // =====================================================
        // STAGE 4: CAPTURE OUTPUTS
        // =====================================================
        stage('📤 Capture Terraform Outputs') {
            steps {
                script {
                    env.INSTANCE_ID = sh(
                        script: "terraform output -raw instance_id",
                        returnStdout: true
                    ).trim()

                    env.INSTANCE_IP = sh(
                        script: "terraform output -raw instance_public_ip",
                        returnStdout: true
                    ).trim()

                    echo "INSTANCE_ID = ${env.INSTANCE_ID}"
                    echo "INSTANCE_IP = ${env.INSTANCE_IP}"
                }
            }
        }

        // =====================================================
        // STAGE 5: DYNAMIC INVENTORY
        // =====================================================
        stage('📄 Generate Dynamic Inventory') {
            steps {
                sh '''
                  cat <<EOF > dynamic_inventory.ini
[splunk]
${INSTANCE_IP} ansible_user=ubuntu
EOF
                '''
            }
        }

        // =====================================================
        // STAGE 6: AWS HEALTH CHECK
        // =====================================================
        stage('🩺 AWS Health Check') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                      export AWS_ACCESS_KEY_ID
                      export AWS_SECRET_ACCESS_KEY

                      aws ec2 wait instance-status-ok \
                        --instance-ids ${INSTANCE_ID} \
                        --region ${AWS_REGION}
                    '''
                }
            }
        }

        // =====================================================
        // STAGE 7: SPLUNK INSTALL
        // =====================================================
        stage('📦 Install Splunk') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ansible-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )
                ]) {
                    sh '''
                      chmod 600 $SSH_KEY
                      ansible-playbook \
                        -i dynamic_inventory.ini \
                        --private-key $SSH_KEY \
                        ansible/splunk.yml
                    '''
                }
            }
        }

        // =====================================================
        // STAGE 8: SPLUNK VERIFICATION
        // =====================================================
        stage('✅ Verify Splunk') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ansible-ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )
                ]) {
                    sh '''
                      chmod 600 $SSH_KEY
                      ansible-playbook \
                        -i dynamic_inventory.ini \
                        --private-key $SSH_KEY \
                        ansible/test-splunk.yml
                    '''
                }
            }
        }

        // =====================================================
        // STAGE 9: VALIDATE DESTROY
        // =====================================================
        stage('🛑 Validate Destroy') {
            steps {
                input message: 'Do you really want to destroy the infrastructure?',
                      ok: 'Yes, Destroy'
            }
        }

        // =====================================================
        // STAGE 10: TERRAFORM DESTROY
        // =====================================================
        stage('🔥 Terraform Destroy') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                      export AWS_ACCESS_KEY_ID
                      export AWS_SECRET_ACCESS_KEY

                      terraform destroy \
                        -auto-approve \
                        -var-file=${BRANCH_NAME}.tfvars
                    '''
                }
            }
        }
    }

    // =====================================================
    // POST ACTIONS
    // =====================================================
    post {
        always {
            echo "🧹 Cleaning up dynamic inventory"
            sh 'rm -f dynamic_inventory.ini'
        }

        failure {
            echo "❌ Pipeline failed — destroying infra"
            withCredentials([
                string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
            ]) {
                sh '''
                  export AWS_ACCESS_KEY_ID
                  export AWS_SECRET_ACCESS_KEY
                  terraform destroy -auto-approve -var-file=${BRANCH_NAME}.tfvars || true
                '''
            }
        }

        aborted {
            echo "⚠️ Pipeline aborted — destroying infra"
            withCredentials([
                string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
            ]) {
                sh '''
                  export AWS_ACCESS_KEY_ID
                  export AWS_SECRET_ACCESS_KEY
                  terraform destroy -auto-approve -var-file=${BRANCH_NAME}.tfvars || true
                '''
            }
        }

        success {
            echo "✅ BYOD-3 Pipeline completed successfully"
        }
    }
}
