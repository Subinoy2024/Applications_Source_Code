pipeline {
  agent { label 'aws-deploy' }

  options { timestamps() }

  stages {

    stage("01 Agent Check") {
      steps {
        sh '''
          echo "Hostname:"
          hostname
          echo "User:"
          whoami
          echo "Workspace:"
          pwd
        '''
      }
    }

    stage("02 Install Tools") {
      steps {
        sh '''
          set -e

          sudo -n apt-get update -y
          sudo -n apt-get install -y git maven curl unzip openssh-client netcat-openbsd ca-certificates

          if ! command -v aws >/dev/null 2>&1; then
            echo "Installing AWS CLI v2..."
            rm -rf aws awscliv2.zip || true
            curl -s https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o awscliv2.zip
            unzip -q awscliv2.zip
            sudo -n ./aws/install --update
          fi

          echo "Tool versions:"
          git --version
          mvn -v | head -n 3
          aws --version
          java -version
        '''
      }
    }

    stage("03 Checkout Verify") {
      steps {
        sh '''
          echo "Repo files:"
          ls -lah
        '''
      }
    }

    stage("04 Unit Test") {
      steps {
        sh '''
          mvn clean test
        '''
      }
    }

    stage("05 Build JAR") {
      steps {
        sh '''
          mvn -DskipTests package
          ls -lh target/*.jar
        '''
      }
    }
  }
}
