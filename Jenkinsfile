pipeline {
  agent any

  environment {
    ANSIBLE_HOST_KEY_CHECKING = 'False'
    INVENTORY = 'ansible/inventory.ini'
    PLAYBOOK  = 'ansible/deploy.yml'
    ARTIFACT_VERSION = readFile('version').trim()
    EMAIL_USERNAME = 'Jenkins'
  }

  stages {
    stage('Tests') {
      when { changeRequest() }
      steps {
        script {
          sh '''
            python3 -m venv venv
            . venv/bin/activate
            pip install -r requirements.txt
            pip install coverage  
            coverage run -m unittest discover tests
            coverage xml -o coverage.xml
            deactivate
          '''
        }
      }
    }

    stage('SonarQube Analysis') {
      when { changeRequest() }
      steps {
        script {
          withSonarQubeEnv('mySonarQube') {
            sh '''
              sonar-scanner \
                -Dsonar.projectKey=cicd-app \
                -Dsonar.sources=. \
                -Dsonar.python.coverage.reportPaths=coverage.xml \
            '''
          }
        }
      }
    }

    stage('SonarQube Quality Gate') {
      when { changeRequest() }
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Approval for Stage') {
      when { changeRequest() }
      steps {
        timeout(time: 1, unit: 'HOURS') {
          input message: 'Deploy to staging?', ok: 'Approve'
        }
      }
    }

    stage('Deploy to stage') {
      when { changeRequest(); }
      steps {
          sh '''
            make deploy-stage
          '''
      }
    }

    // stage('Create Docker image') {
    //   when { branch 'master' }
    //   steps {
    //       sh "docker build -t cicd-app:${ARTIFACT_VERSION} ."
    //   }
    // }

    stage('Approval for Production') {
      when { branch 'master' }
      steps {
        timeout(time: 1, unit: 'HOURS') {
          input message: 'Deploy to production?', ok: 'Approve'
        }
      }
    }

    stage('Build Docker image') {
      when { branch 'master' }
      steps {
        sh "TAG=${ARTIFACT_VERSION} make build"
      }
    }

    stage('Push Docker image to Docker Hub') {
      when { branch 'master' }
      steps {
        script {    
          withCredentials([usernamePassword(
              credentialsId: 'DOCKERHUB',
              usernameVariable: 'USERNAME',
              passwordVariable: 'PASSWORD'
          )]) {
            sh "TAG=${ARTIFACT_VERSION} make push-latest"
          }
        }
      }
    }

    stage('Deploy to production') {
      when { branch 'master' }
      steps {
        withCredentials([string(credentialsId: 'ansible-vault-pass', variable: 'VAULTPASS')]) {

          sh '''
            # create secure temp file
            KCFG=$(mktemp)
            chmod 600 "$KCFG"

            # ensure cleanup on exit
            trap "rm -f $KCFG" EXIT

            # decrypt vault to the temp file
            ansible-vault view ansible/kubeconfig.vault \
              --vault-password-file=<(echo "$VAULTPASS") \
              > "$KCFG"

            kubectl --kubeconfig="$KCFG" rollout restart deployment cicd-app
          '''
        }
      }
    }
  }

post {
    success {
        emailext(
            subject: "SUCCESS: Build ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: "Build succeeded.\n${env.BUILD_URL}\nVersion: ${ARTIFACT_VERSION}\nCommit hash: ${GIT_COMMIT}\nBranch: ${GIT_BRANCH}",
            to: '$DEFAULT_RECIPIENTS',
            from: "${EMAIL_USERNAME}"
        )
        echo "Deployment finished: SUCCESS"
    }

    failure {
        emailext(
            subject: "FAILURE: Build ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: "Build failed.\n${env.BUILD_URL}\nVersion: ${ARTIFACT_VERSION}\nCommit hash: ${GIT_COMMIT}\nBranch: ${GIT_BRANCH}",
            to: '$DEFAULT_RECIPIENTS',
            from: "${EMAIL_USERNAME}"
        )
        echo "Deployment finished: FAILURE"
    }

    aborted {
        emailext(
            subject: "ABORTED: Build ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: "Build was manually aborted.\n${env.BUILD_URL}\nVersion: ${ARTIFACT_VERSION}\nCommit hash: ${GIT_COMMIT}\nBranch: ${GIT_BRANCH}",
            to: '$DEFAULT_RECIPIENTS',
            from: "${EMAIL_USERNAME}"
        )
        echo "Deployment finished: ABORTED"
    }

    unstable {
        emailext(
            subject: "UNSTABLE: Build ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: "Build is unstable.\n${env.BUILD_URL}\nVersion: ${ARTIFACT_VERSION}\nCommit hash: ${GIT_COMMIT}\nBranch: ${GIT_BRANCH}",
            to: '$DEFAULT_RECIPIENTS',
            from: "${EMAIL_USERNAME}"
        )
        echo "Deployment finished: UNSTABLE"
    }
}


}
