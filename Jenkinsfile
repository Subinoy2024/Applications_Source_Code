pipeline {
  agent { label 'aws-deploy' }
  options { timestamps() }

  environment {
    CFN_TEMPLATE = "cfn/ec2-nginx-app.yml"
    JAR_GLOB     = "target/*.jar"
    AWS_REGION   = "${params.AWS_REGION ?: 'ap-south-1'}"
    STACK_NAME   = "${params.STACK_NAME ?: 'petclinic-stack'}"
  }

  parameters {
    string(name: 'AWS_REGION', defaultValue: 'ap-south-1', description: 'AWS region')
    string(name: 'STACK_NAME', defaultValue: 'petclinic-stack', description: 'CloudFormation stack name')

    string(name: 'VpcCidr', defaultValue: '10.20.0.0/16', description: 'VPC CIDR')
    string(name: 'PublicSubnetCidr', defaultValue: '10.20.0.0/24', description: 'Public subnet CIDR')
    string(name: 'AllowedSshCidr', defaultValue: '0.0.0.0/0', description: 'SSH allowed CIDR')

    string(name: 'KeyName', defaultValue: 'aws_subinoy_ind', description: 'EC2 KeyPair name')
    string(name: 'AmiId', defaultValue: 'ami-0ff5003538b60d5ec', description: 'Ubuntu AMI ID')
  }

  stages {

    stage("01 Checkout") {
      steps {
        checkout scm
        sh 'ls -lah'
      }
    }

    stage("02 Install Tools") {
      steps {
        sh '''#!/usr/bin/env bash
          set -e
          sudo apt-get update -y
          sudo apt-get install -y git maven curl unzip openssh-client netcat-openbsd ca-certificates jq
          if ! command -v aws >/dev/null 2>&1; then
            curl -sS https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o awscliv2.zip
            unzip -q awscliv2.zip
            sudo ./aws/install --update
          fi
          aws --version
        '''
      }
    }

    stage("03 AWS Auth Validate") {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'aws-creds',
          usernameVariable: 'AWS_ACCESS_KEY_ID',
          passwordVariable: 'AWS_SECRET_ACCESS_KEY'
        )]) {
          sh '''#!/usr/bin/env bash
            set -e
            export AWS_DEFAULT_REGION="${AWS_REGION}"
            aws sts get-caller-identity
          '''
        }
      }
    }

    stage("04 Unit Test") {
      steps {
        sh '''#!/usr/bin/env bash
          set -e
          mvn -q clean test
        '''
      }
    }

    stage("05 Build JAR") {
      steps {
        sh '''#!/usr/bin/env bash
          set -e
          mvn -q -DskipTests package
          ls -lh target/*.jar
        '''
      }
    }

    stage("06 CloudFormation Deploy") {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'aws-creds',
          usernameVariable: 'AWS_ACCESS_KEY_ID',
          passwordVariable: 'AWS_SECRET_ACCESS_KEY'
        )]) {
          sh '''#!/usr/bin/env bash
            set -e
            export AWS_DEFAULT_REGION="${AWS_REGION}"

            STACK_STATUS=$(aws cloudformation describe-stacks \
              --stack-name "${STACK_NAME}" \
              --query "Stacks[0].StackStatus" \
              --output text 2>/dev/null || echo "NOT_FOUND")

            if [ "$STACK_STATUS" = "ROLLBACK_COMPLETE" ]; then
              aws cloudformation delete-stack --stack-name "${STACK_NAME}"
              aws cloudformation wait stack-delete-complete --stack-name "${STACK_NAME}"
            fi

            aws cloudformation deploy \
              --stack-name "${STACK_NAME}" \
              --template-file "${CFN_TEMPLATE}" \
              --capabilities CAPABILITY_NAMED_IAM \
              --no-fail-on-empty-changeset \
              --parameter-overrides \
                VpcCidr="${VpcCidr}" \
                PublicSubnetCidr="${PublicSubnetCidr}" \
                AllowedSshCidr="${AllowedSshCidr}" \
                KeyName="${KeyName}" \
                AmiId="${AmiId}"
          '''
        }
      }
    }

    stage("07 Get EC2 Public IP") {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'aws-creds',
          usernameVariable: 'AWS_ACCESS_KEY_ID',
          passwordVariable: 'AWS_SECRET_ACCESS_KEY'
        )]) {
          script {
            env.EC2_PUBLIC_IP = sh(
              script: '''#!/usr/bin/env bash
                export AWS_DEFAULT_REGION="${AWS_REGION}"
                aws cloudformation describe-stacks \
                  --stack-name "${STACK_NAME}" \
                  --query "Stacks[0].Outputs[?OutputKey=='InstancePublicIp'].OutputValue" \
                  --output text
              ''',
              returnStdout: true
            ).trim()
            echo "EC2 Public IP: ${env.EC2_PUBLIC_IP}"
          }
        }
      }
    }

    stage("08 Wait for SSH") {
      steps {
        sh '''#!/usr/bin/env bash
          set -e
          for i in {1..60}; do
            nc -z "${EC2_PUBLIC_IP}" 22 && exit 0
            sleep 5
          done
          exit 1
        '''
      }
    }

    stage("09 Deploy JAR to EC2") {
      steps {
        withCredentials([sshUserPrivateKey(
          credentialsId: 'ec2-ssh-key',
          keyFileVariable: 'SSH_KEY_FILE'
        )]) {
          sh '''#!/usr/bin/env bash
            set -e
            chmod 600 "${SSH_KEY_FILE}"
            SSH_USER="ubuntu"

            JAR_FILE=$(ls -1 target/*.jar | head -n 1)

            scp -o StrictHostKeyChecking=no \
              -i "${SSH_KEY_FILE}" \
              "${JAR_FILE}" \
              "${SSH_USER}@${EC2_PUBLIC_IP}:/tmp/petclinic.jar"

            ssh -o StrictHostKeyChecking=no \
              -i "${SSH_KEY_FILE}" \
              "${SSH_USER}@${EC2_PUBLIC_IP}" <<'EOF'
              sudo mkdir -p /opt/petclinic
              sudo mv /tmp/petclinic.jar /opt/petclinic/petclinic.jar
              sudo chown -R petclinic:petclinic /opt/petclinic
              sudo systemctl restart petclinic
              sudo systemctl status petclinic --no-pager | head -n 20
EOF
          '''
        }
      }
    }

    stage("10 Health Check") {
      steps {
        sh '''#!/usr/bin/env bash
          set -e
          for i in {1..30}; do
            code=$(curl -s -o /dev/null -w "%{http_code}" http://${EC2_PUBLIC_IP}/ || true)
            [ "$code" = "200" ] || [ "$code" = "302" ] && exit 0
            sleep 5
          done
          exit 1
        '''
      }
    }
  }

  post {
    always {
      echo "Pipeline finished. App URL: http://${env.EC2_PUBLIC_IP}/"
    }
  }
}
