pipeline {
  agent { label 'aws-deploy' }

  options { timestamps() }

  stages {

    stage("01 Checkout") {
      steps {
        checkout scm
        // Safety cleanup in case a previous build left these in workspace
        sh 'rm -rf aws awscliv2.zip || true'
      }
    }

    stage("02 Install Tools") {
      steps {
        sh '''
          set -e

          sudo -n apt-get update -y
          sudo -n apt-get install -y \
            git maven curl unzip openssh-client netcat-openbsd ca-certificates

          # Install AWS CLI v2 OUTSIDE the Jenkins workspace (/tmp) so Maven/Checkstyle won't scan it
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

    stage("03 Workspace Verify") {
      steps {
        sh '''
          echo "Repo files:"
          ls -lah
          echo "Confirm no aws folder in workspace:"
          test ! -d aws && echo "OK: aws/ not present"
        '''
      }
    }

    stage("04 Unit Test") {
      steps {
        sh '''
          set -e
          mvn clean test
        '''
      }
    }

    stage("05 Build JAR") {
      steps {
        sh '''
          set -e
          mvn -DskipTests package
          ls -lh target/*.jar
        '''
      }
    }

    stage("06 Archive Artifact") {
      steps {
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
      }
    }

  }

  post {
    always {
      // Extra cleanup to avoid future workspace pollution
      sh 'rm -rf aws awscliv2.zip || true'
    }
  }
}
