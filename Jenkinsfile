pipeline {
  agent { label 'aws-deploy' }
  options { timestamps() }

  environment {
    // Artifact path used in deploy stage
    JAR_GLOB = "target/*.jar"
    CFN_TEMPLATE = "cfn/ec2-nginx-app.yml"
  }

  stages {

    stage("01 Checkout") {
      steps {
        checkout scm
        sh 'rm -rf aws awscliv2.zip || true'
      }
    }

    stage("02 Install Tools") {
      steps {
        sh '''
          set -e
          sudo -n apt-get update -y
          sudo -n apt-get install -y git maven curl unzip openssh-client netcat-openbsd ca-certificates jq

          if ! command -v aws >/dev/null 2>&1; then
            echo "Installing AWS CLI v2 (outside workspace)..."
            TMP_DIR="$(mktemp -d)"
            cd "$TMP_DIR"
            curl -s https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o awscliv2.zip
            unzip -q awscliv2.zip
            sudo -n ./aws/install --update
            cd -
            rm -rf "$TMP_DIR"
          fi

          echo "Tool versions:"
          git --version
          mvn -v | head -n 3
          aws --version
          java -version
        '''
      }
    }

    stage("03 Unit Test") {
      steps {
        sh '''
          set -e
          mvn clean test
        '''
      }
    }

    stage("04 Build JAR") {
      steps {
        sh '''
          set -e
          mvn -DskipTests package
          ls -lh target/*.jar
        '''
      }
    }

    stage("05 AWS Auth Validate") {
      steps {
        withCredentials([usernamePassword(credentialsId: 'aws-creds',
          usernameVariable: 'AWS_ACCESS_KEY_ID',
          passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {

          sh '''
            set -e
            export AWS_DEFAULT_REGION="${AWS_REGION:-ap-south-1}"
            aws sts get-caller-identity
          '''
        }
      }
    }

    stage("06 CloudFormation Deploy (Create/Update EC2)") {
      steps {
        withCredentials([usernamePassword(credentialsId: 'aws-creds',
          usernameVariable: 'AWS_ACCESS_KEY_ID',
          passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {

          sh '''
            set -e
            export AWS_DEFAULT_REGION="${AWS_REGION:-ap-south-1}"

            test -f "${CFN_TEMPLATE}" || (echo "Missing template: ${CFN_TEMPLATE}" && exit 1)

            aws cloudformation deploy \
              --stack-name "${STACK_NAME:-petclinic-stack}" \
              --template-file "${CFN_TEMPLATE}" \
              --capabilities CAPABILITY_NAMED_IAM \
              --parameter-overrides \
                VpcId="${VPC_ID}" \
                SubnetId="${SUBNET_ID}" \
                KeyName="${KEY_NAME}" \
                AmiId="${AMI_ID}" \
                AllowedSshCidr="${ALLOWED_SSH_CIDR:-0.0.0.0/0}"

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

          sh '''
            set -e
            export AWS_DEFAULT_REGION="${AWS_REGION:-ap-south-1}"

            APP_IP=$(aws cloudformation describe-stacks \
              --stack-name "${STACK_NAME:-petclinic-stack}" \
              --query "Stacks[0].Outputs[?OutputKey=='AppPublicIp'].OutputValue" \
              --output text)

            echo "EC2 Public IP: $APP_IP"
            test -n "$APP_IP"

            echo "$APP_IP" > ec2_ip.txt
          '''
        }
      }
    }

    stage("08 Wait for SSH") {
      steps {
        sh '''
          set -e
          APP_IP=$(cat ec2_ip.txt)
          echo "Waiting for SSH on $APP_IP..."
          for i in $(seq 1 60); do
            if nc -z "$APP_IP" 22; then
              echo "SSH is up."
              exit 0
            fi
            sleep 5
          done
          echo "ERROR: SSH not reachable after timeout"
          exit 1
        '''
      }
    }

    stage("09 Deploy JAR to EC2 + Restart") {
      steps {
        script {
          def jarFile = sh(script: "ls -1 ${env.JAR_GLOB} | head -n 1", returnStdout: true).trim()
          if (!jarFile) { error("JAR not found in target/. Build failed?") }
          env.JAR_FILE = jarFile
        }

        sshagent(credentials: ['ec2-ssh-key']) {
          sh '''
            set -e
            APP_IP=$(cat ec2_ip.txt)

            echo "Deploying ${JAR_FILE} to EC2..."
            scp -o StrictHostKeyChecking=no "${JAR_FILE}" ubuntu@"$APP_IP":/tmp/petclinic.jar

            echo "Move jar + restart service..."
            ssh -o StrictHostKeyChecking=no ubuntu@"$APP_IP" <<'EOF'
              set -e
              sudo mkdir -p /opt/petclinic
              sudo mv /tmp/petclinic.jar /opt/petclinic/petclinic.jar
              sudo chown -R petclinic:petclinic /opt/petclinic
              sudo systemctl daemon-reload || true
              sudo systemctl restart petclinic
              sudo systemctl status petclinic --no-pager -l | head -n 20
EOF
          '''
        }
      }
    }

    stage("10 Health Check (Nginx 80)") {
      steps {
        sh '''
          set -e
          APP_IP=$(cat ec2_ip.txt)
          echo "Testing: http://$APP_IP/"
          for i in $(seq 1 30); do
            if curl -fsS "http://$APP_IP/" >/dev/null; then
              echo "SUCCESS: App is reachable via Nginx"
              exit 0
            fi
            sleep 5
          done
          echo "ERROR: App not reachable"
          exit 1
        '''
      }
    }

  }

  post {
    always {
      archiveArtifacts artifacts: "ec2_ip.txt", allowEmptyArchive: true
      sh 'rm -rf aws awscliv2.zip || true'
    }
  }
}
