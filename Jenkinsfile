pipeline {
  agent { label 'aws-deploy' }
  options { timestamps() }

  environment {
    CFN_TEMPLATE = "cfn/ec2-nginx-app.yml"
    JAR_GLOB     = "target/*.jar"
    STACK_NAME   = "petclinic-stack"
    AWS_REGION   = "${params.AWS_REGION ?: 'ap-south-1'}"
  }

  parameters {
    string(name: 'AWS_REGION', defaultValue: 'ap-south-1', description: 'AWS region')
    string(name: 'VpcCidr', defaultValue: '192.168.0.0/22', description: 'New VPC CIDR')
    string(name: 'PublicSubnetCidr', defaultValue: '192.168.0.0/22', description: 'Public subnet CIDR')
    string(name: 'AllowedSshCidr', defaultValue: '0.0.0.0/0', description: 'SSH allowed CIDR (use your public IP/32 ideally)')
    string(name: 'KeyName', defaultValue: 'aws_subinoy', description: 'Existing EC2 KeyPair name')
    string(name: 'AmiId', defaultValue: '', description: 'Ubuntu AMI ID for this region (required)')
    string(name: 'InstanceType', defaultValue: 't2.micro', description: 'EC2 instance type')
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
        sh '''
          set -e
          sudo -n apt-get update -y
          sudo -n apt-get install -y git maven curl unzip openssh-client netcat-openbsd ca-certificates jq

          # AWS CLI v2 install in /tmp (no workspace pollution)
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
          sh '''
            set -e
            export AWS_DEFAULT_REGION="${AWS_REGION}"
            aws sts get-caller-identity
          '''
        }
      }
    }

    stage("04 Unit Test") {
      steps {
        sh 'mvn -q clean test'
      }
    }

    stage("05 Build JAR") {
      steps {
        sh '''
          set -e
          mvn -q -DskipTests package
          ls -lh target/*.jar
        '''
      }
    }

    stage("06 CloudFormation Deploy (Create VPC+Subnet+EC2)") {
      steps {
        withCredentials([usernamePassword(credentialsId: 'aws-creds',
                          usernameVariable: 'AWS_ACCESS_KEY_ID',
                          passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          sh '''
            set -e
            export AWS_DEFAULT_REGION="${AWS_REGION}"

            test -f "${CFN_TEMPLATE}" || (echo "Missing template: ${CFN_TEMPLATE}" && exit 1)
            test -n "${AmiId}" || (echo "AmiId parameter is required (Ubuntu AMI)" && exit 1)

            aws cloudformation deploy \
              --stack-name "${STACK_NAME}" \
              --template-file "${CFN_TEMPLATE}" \
              --capabilities CAPABILITY_NAMED_IAM \
              --parameter-overrides \
                VpcCidr="${VpcCidr}" \
                PublicSubnetCidr="${PublicSubnetCidr}" \
                AllowedSshCidr="${AllowedSshCidr}" \
                KeyName="${KeyName}" \
                AmiId="${AmiId}" \
                InstanceType="${InstanceType}"

            echo "Stack deployed."
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
              script: '''
                set -e
                export AWS_DEFAULT_REGION="${AWS_REGION}"
                aws cloudformation describe-stacks --stack-name "${STACK_NAME}" \
                  --query "Stacks[0].Outputs[?OutputKey=='InstancePublicIp'].OutputValue" --output text
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
        sh '''
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
        // ec2-ssh-key: add your aws_subinoy.pem here (Jenkins Credentials)
        withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key',
                          keyFileVariable: 'SSH_KEY_FILE',
                          usernameVariable: 'SSH_USER')]) {
          sh '''
            set -e
            JAR_FILE="$(ls -1 ${JAR_GLOB} | head -n 1)"
            echo "Deploying: ${JAR_FILE}"

            chmod 600 "${SSH_KEY_FILE}"

            # Copy jar
            scp -o StrictHostKeyChecking=no -i "${SSH_KEY_FILE}" \
              "${JAR_FILE}" "${SSH_USER}@${EC2_PUBLIC_IP}:/tmp/petclinic.jar"

            # Move + permissions + restart
            ssh -o StrictHostKeyChecking=no -i "${SSH_KEY_FILE}" "${SSH_USER}@${EC2_PUBLIC_IP}" <<'EOF'
              set -e
              sudo mv /tmp/petclinic.jar /opt/petclinic/petclinic.jar
              sudo chown petclinic:petclinic /opt/petclinic/petclinic.jar
              sudo systemctl restart petclinic
              sudo systemctl --no-pager --full status petclinic | head -n 30
EOF
          '''
        }
      }
    }

    stage("10 Health Check") {
      steps {
        sh '''
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
