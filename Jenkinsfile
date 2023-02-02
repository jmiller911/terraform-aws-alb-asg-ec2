pipeline {
    agent {  label 'master' }
    environment {
        SSM = "/demo/terraform/${AWS_ENV}"
    }
    parameters {
        choice choices: ['dev', 'uat', 'prd'], description: 'Select an AWS account', name: 'AWS_ENV'
        string(name: 'VPC_NAME', defaultValue: '', description: 'Enter name for VPC')
        choice choices: ['us-east-1', 'us-east-2', 'us-west-1'], description: 'Select an AWS region', name: 'AWS_REGION'
        choice choices: ['apply', 'destroy'], description: 'Deploy or destroy', name: 'TERRAFORM'
    }
    stages {
        stage("Set Environment") {
            steps {
                script {
                    switch(env.AWS_ENV) {
                    case 'dev':
                        env.TF_STATE_BUCKET="demo-tf-907207106954-us-east-1"
                        env.TF_STATE_OBJECT_KEY="terraform.tfstate"
                        env.TF_LOCK_DB="demo-tf-lock-table"
                        env.AWS_REGION="us-east-1"
                        break
                    case 'uat':
                        env.TF_STATE_BUCKET="demo-tf-907207106954-us-east-1"
                        env.TF_STATE_OBJECT_KEY="terraform.tfstate"
                        env.TF_LOCK_DB="demo-tf-lock-table"
                        env.AWS_REGION="us-east-1"
                        break
                    case 'prd':
                        env.TF_STATE_BUCKET="demo-tf-907207106954-us-east-1"
                        env.TF_STATE_OBJECT_KEY="terraform.tfstate"
                        env.TF_LOCK_DB="demo-tf-lock-table"
                        env.AWS_REGION="us-east-1"
                        break     
                    }
                }
            }
        }
        stage('Install Terraform') {
            steps {
                // dir('.tfenv') {
                //     checkout([$class: 'GitSCM', branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: '', url: 'https://github.com/tfutils/tfenv.git']]])
                // }
                sh '''
                    rm -rf .tfenv
                    git clone --depth=1 https://github.com/tfutils/tfenv.git .tfenv
                    echo 'export PATH="$WORKSPACE/.tfenv/bin:$PATH"' > ~/.bash_profile
                    . ~/.bash_profile
                    tfenv install latest
                    tfenv use latest
                    terraform --version
                '''
            }
        }

        stage('Create TFVARS File') {
            steps {
                withAWSParameterStore(credentialsId: 'aws_keys', naming: 'basename', path: "${SSM}", recursive: true, regionName: "${AWS_REGION}") {
                    sh '''
                        rm -rf terraform_${AWS_ENV}.tfvars .terraform
cat << TFVARS > ./terraform_${AWS_ENV}.tfvars
cidr = "${CIDR}"
private_subnets = ${PRIVATE_SUBNETS}
public_subnets = ${PUBLIC_SUBNETS}
TFVARS
cat terraform_${AWS_ENV}.tfvars
                    '''
                }
                post {
                    success {
                        stash name: "tfvars", includes: "terraform_${AWS_ENV}.tfvars"
                    }
                }
            }
        }

        stage('Deploy Demo App') {
             when {
                environment name: 'TERRAFORM', value: 'apply'
            }
            steps {
                unstash "tfvars"
//                 withAWSParameterStore(credentialsId: 'aws_keys', naming: 'basename', path: "${SSM}", recursive: true, regionName: "${AWS_REGION}") {
//                     sh '''
//                         . ~/.bash_profile
//                         rm -rf terraform_${AWS_ENV}.tfvars .terraform
// cat << TFVARS > ./terraform_${AWS_ENV}.tfvars
// cidr = "${CIDR}"
// private_subnets = ${PRIVATE_SUBNETS}
// public_subnets = ${PUBLIC_SUBNETS}
// TFVARS
// cat terraform_${AWS_ENV}.tfvars
//                     '''
//                 }
                withAWS(credentials:'aws_keys', region: "${AWS_REGION}") {
                    sh """
                        cat terraform_${AWS_ENV}.tfvars
                        export TF_STATE_BUCKET="${TF_STATE_BUCKET}"
                        export TF_STATE_OBJECT_KEY="${TF_STATE_OBJECT_KEY}"
                        export TF_LOCK_DB="${TF_LOCK_DB}"
                        echo "${TF_STATE_BUCKET}"
                        . ~/.bash_profile
                        terraform init \
                        -backend=true \
                        -backend-config key="${TF_STATE_OBJECT_KEY}" \
                        -backend-config bucket="${TF_STATE_BUCKET}" \
                        -backend-config dynamodb_table="${TF_LOCK_DB}" \
                        -force-copy
                        terraform workspace select ${AWS_ENV} || terraform workspace new ${AWS_ENV}
                        terraform plan -var-file=terraform_${AWS_ENV}.tfvars -out=tfplan -input=false
                        terraform apply --auto-approve tfplan
                    """
                }
            }
        }

        stage('Destroy Demo App') {
             when {
                environment name: 'TERRAFORM', value: 'destroy'
            }
            steps {
                unstash "tfvars"
//                 withAWSParameterStore(credentialsId: 'aws_keys', naming: 'basename', path: "${SSM}", recursive: true, regionName: "${AWS_REGION}") {
//                     sh '''
//                         cd aws-resources/create-vpc
//                         . ~/.bash_profile
//                         rm -rf terraform_${AWS_ENV}.tfvars .terraform

// cat << TFVARS > ./terraform_${AWS_ENV}.tfvars
// name = "${VPC_NAME}"
// cidr = "${CIDR}"
// private_subnets = ${PRIVATE_SUBNETS}
// public_subnets = ${PUBLIC_SUBNETS}
// database_subnets = ${DATABASE_SUBNETS}
// elasticache_subnets = ${ELASTICACHE_SUBNETS}
// intra_subnets = ${INTRANET_SUBNETS}
// TFVARS
                       
//                     '''
//                 }
                withAWS(roleAccount: "${AWS_ACCT}", role: "${AWS_ROLE}", region: "${AWS_REGION}") {
                    sh '''
                        . ~/.bash_profile
                        terraform init -backend-config=backend-${AWS_ENV}.tfvars -force-copy
                        terraform workspace select ${AWS_ENV} || terraform workspace new ${AWS_ENV}
                        terraform destroy -var-file=terraform_${AWS_ENV}.tfvars --auto-approve
                    '''
                }
            }
        }
    }
}
