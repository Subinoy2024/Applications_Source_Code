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

    string(name: 'VpcCidr', defaultValue: '10.20.0.0/16', description: 'New VPC CIDR (RFC1918 private range)')
    string(name: 'PublicSubnetCidr', defaultValue: '10.20.0.0/24', description: 'Public subnet CIDR (inside VPC)')
    string(name: 'AllowedSshCidr', defaultValue: '0.0.0.0/0', description: 'SSH allowed CIDR (use your public IP/32 ideally)')

    string(name: 'KeyName', defaultValue: 'aws_subinoy_ind', description: 'Existing EC2 KeyPair name (region-specific)')
    string(name: 'AmiId', defaultValue: 'ami-0ff5003538b60d5ec', description: 'AMI ID for this region (required)')
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
          sudo -n apt-get update -y
          sudo -n apt-get install -y git maven curl unzip openssh-client netcat-openbsd ca-certificates jq

          # AWS CLI v2 install (only if missing)
          if ! command -v aws >/dev/null 2>&1; then
            echo "Installing AWS CLI v2..."
            tmpdir="$(mktemp -d)"
            cd "$tmpdir"
            curl -sS https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o awscliv2.zip
            unzip -q awscliv2.zip
            sudo -n ./aws/install --update
            cd /
            rm -rf "$tmpdir"
          fi

          aws --version
          mvn -v | head -n 3
        '''
      }
    }

    stage("03 AWS Auth Validate") {
      steps {
        withCredentials([usernamePassword(credentialsId: 'aws-creds',
                          usernameVariable: 'AWS_ACCESS_KEY_ID',
                          passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
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

    stage("06 CloudFormation Deploy (Auto-heal ROLLBACK_COMPLETE)") {
      steps {
        withCredentials([usernamePassword(credentialsId: 'aws-creds',
                          usernameVariable: 'AWS_ACCESS_KEY_ID',
                          passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          sh '''#!/usr/bin/env bash
            set -euo pipefail
            export AWS_DEFAULT_REGION="${AWS_REGION}"

            test -f "${CFN_TEMPLATE}" || (echo "Missing template: ${CFN_TEMPLATE}" && exit 1)
            test -n "${AmiId}" || (echo "AmiId parameter is required" && exit 1)

            STACK_STATUS=$(aws cloudformation describe-stacks \
              --stack-name "${STACK_NAME}" \
              --query "Stacks[0].StackStatus" \
              --output text 2>/dev/null || echo "NOT_FOUND")

            echo "Current stack status: ${STACK_STATUS}"

            if [ "${STACK_STATUS}" = "ROLLBACK_COMPLETE" ]; then
              echo "Stack is in ROLLBACK_COMPLETE. Deleting stack ${STACK_NAME}..."
              aws cloudformation delete-stack --stack-name "${STACK_NAME}"
              aws cloudformation wait stack-delete-complete --stack-name "${STACK_NAME}"
              echo "Old stack deleted successfully."
            fi

            # Do NOT pass InstanceType; template Default+AllowedValues controls it.
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
                AmiId="${AmiId}" \
            || {
              echo "CloudFormation failed."

              if aws cloudformation describe-stacks --stack-name "${STACK_NAME}" >/dev/null 2>&1; then
                aws cloudformation describe-stack-events --stack-name "${STACK_NAME}" \
                  --query "StackEvents[?ResourceStatus=='CREATE_FAILED' || ResourceStatus=='UPDATE_FAILED'].[Timestamp,LogicalResourceId,ResourceStatusReason]" \
                  --output table || true
              else
                echo "No stack exists yet (deploy failed before stack creation)."
              fi

              exit 1
            }

            echo "Stack deployed."
            aws cloudformation describe-stacks --stack-name "${STACK_NAME}" \
              --query "Stacks[0].Outputs" --output table || true
          '''
        }
      }
    }

    stage("07 Get EC2 Public IP") {
      steps {
        withCredentials([usernamePassword(credentialsId: 'aws-creds',
                          usernameVariable: 'AWS_ACCESS_KEY_ID',
                          passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          script {
            def ip = sh(
              script: '''#!/usr/bin/env bash
                set -e
                export AWS_DEFAULT_REGION="${AWS_REGION}"
                aws cloudformation describe-stacks --stack-name "${STACK_NAME}" \
                  --query "Stacks[0].Outputs[?OutputKey=='InstancePublicIp'].OutputValue" \
                  --output text
              ''',
              returnStdout: true
            ).trim()

            if (!ip) { error("Could not read InstancePublicIp from stack outputs") }
            env.EC2_PUBLIC_IP = ip
            echo "EC2 Public IP: ${env.EC2_PUBLIC_IP}"
          }
        }
      }
    }

    stage("08 Wait for SSH") {
      steps {
        sh '''#!/usr/bin/env bash
          set -e
          echo "Waiting for SSH on ${EC2_PUBLIC_IP}:22 ..."
          for i in $(seq 1 60); do
            if nc -z "${EC2_PUBLIC_IP}" 22; then
              echo "SSH port is open."
              exit 0
            fi
            sleep 5
          done
          echo "Timeout waiting for SSH"
          exit 1
        '''
      }
    }

    stage("09 Deploy JAR to EC2 + Restart") {
      steps {
        // IMPORTANT: Update Jenkins credential ec2-ssh-key to use aws_subinoy_ind.pem and username ec2-user
        withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key',
                          keyFileVariable: 'SSH_KEY_FILE')]) {
          sh '''#!/usr/bin/env bash
            set -e
            JAR_FILE="$(ls -1 ${JAR_GLOB} | head -n 1)"
            echo "Deploying: ${JAR_FILE}"

            chmod 600 "${SSH_KEY_FILE}"

            SSH_LOGIN_USER="ec2-user"

            scp -o StrictHostKeyChecking=no -i "${SSH_KEY_FILE}" \
              "${JAR_FILE}" "${SSH_LOGIN_USER}@${EC2_PUBLIC_IP}:/tmp/petclinic.jar"

            ssh -o StrictHostKeyChecking=no -i "${SSH_KEY_FILE}" "${SSH_LOGIN_USER}@${EC2_PUBLIC_IP}" <<'EOF'
              set -e
              sudo mkdir -p /opt/petclinic
              sudo mv /tmp/petclinic.jar /opt/petclinic/petclinic.jar
              sudo chown -R petclinic:petclinic /opt/petclinic || true
              sudo chown petclinic:petclinic /opt/petclinic/petclinic.jar
              sudo systemctl restart petclinic
              sudo systemctl --no-pager --full status petclinic | head -n 40
EOF
          '''
        }
      }
    }

    stage("10 Health Check") {
      steps {
        sh '''#!/usr/bin/env bash
          set -e
          echo "Checking http://${EC2_PUBLIC_IP}/ ..."
          for i in $(seq 1 30); do
            code=$(curl -s -o /dev/null -w "%{http_code}" "http://${EC2_PUBLIC_IP}/" || true)
            if [ "$code" = "200" ] || [ "$code" = "302" ]; then
              echo "SUCCESS: App reachable (HTTP $code)"
              exit 0
            fi
            sleep 5
          done
          echo "FAILED: App not reachable"
          exit 1
        '''
      }
    }
  }

  post {
    always {
      echo "Pipeline finished. If success, open: http://${env.EC2_PUBLIC_IP}/"
    }
  }
}
