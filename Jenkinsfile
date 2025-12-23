pipeline {
  agent any

  environment {
    TF_IN_AUTOMATION = '1'
    TF_CLI_ARGS = '-no-color'
  }

  stages {

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

    stage('Inspect TFVARS') {
      steps {
        sh '''
          echo "Inspecting tfvars for branch: $BRANCH_NAME"
          if [ -f ${BRANCH_NAME}.tfvars ]; then
            cat ${BRANCH_NAME}.tfvars
          else
            echo "ERROR: ${BRANCH_NAME}.tfvars not found"
            exit 1
          fi
        '''
      }
    }

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
            ls -l *.tfvars

            terraform plan \
              -var-file=${BRANCH_NAME}.tfvars \
              -out=plan-${BRANCH_NAME}.tfplan

            terraform show -no-color plan-${BRANCH_NAME}.tfplan
          '''
        }
      }
    }

    stage('Validate & Apply') {
      when {
        branch 'dev'
      }
      steps {
        input message: 'Approve apply to dev?', ok: 'Apply'

        withCredentials([
          string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
          string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
        ]) {
          sh '''
            export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
            export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY

            echo "Applying Terraform plan for branch: $BRANCH_NAME"
            ls -l plan-*.tfplan

            terraform apply -auto-approve plan-${BRANCH_NAME}.tfplan
          '''
        }
      }
    }
  }
}
