pipeline {
  agent any

  environment {
    TF_IN_AUTOMATION = '1'
    TF_CLI_ARGS = '-no-color'
  }

  stages {

    stage('Init') {
      steps {
        script {
          def branch = env.BRANCH_NAME ?: env.GIT_BRANCH ?: 'dev'

          withCredentials([
            string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
            string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY'),
            sshUserPrivateKey(
              credentialsId: 'ansible-ssh-key',
              keyFileVariable: 'SSH_KEY',
              usernameVariable: 'SSH_USER'
            )
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
    }

    stage('Inspect TFVARS') {
      steps {
        script {
          def branch = env.BRANCH_NAME ?: env.GIT_BRANCH ?: 'dev'
          sh """
            echo "Displaying vars for branch: ${branch}"
            if [ -f ${branch}.tfvars ]; then
              cat ${branch}.tfvars
            else
              echo "${branch}.tfvars not found"
            fi
          """
        }
      }
    }

    stage('Plan') {
      steps {
        script {
          def branch = env.BRANCH_NAME ?: env.GIT_BRANCH ?: 'dev'

          withCredentials([
            string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
            string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY'),
            sshUserPrivateKey(
              credentialsId: 'ansible-ssh-key',
              keyFileVariable: 'SSH_KEY',
              usernameVariable: 'SSH_USER'
            )
          ]) {
            sh '''
              export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
              export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY

              terraform plan \
                -var-file=${branch}.tfvars \
                -out=plan-${branch}.tfplan

              terraform show -no-color plan-${branch}.tfplan
            '''
          }
        }
      }
    }

    stage('Validate & Apply') {
      when {
        branch 'dev'
      }
      steps {
        script {
          input message: 'Approve apply to dev?', ok: 'Apply'

          def branch = env.BRANCH_NAME ?: env.GIT_BRANCH ?: 'dev'

          withCredentials([
            string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
            string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY'),
            sshUserPrivateKey(
              credentialsId: 'ansible-ssh-key',
              keyFileVariable: 'SSH_KEY',
              usernameVariable: 'SSH_USER'
            )
          ]) {
            sh '''
              export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
              export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY

              terraform apply -auto-approve plan-${branch}.tfplan
            '''
          }
        }
      }
    }
  }
}
