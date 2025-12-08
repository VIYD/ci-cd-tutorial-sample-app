pipeline {
  agent any

  environment {
    ANSIBLE_HOST_KEY_CHECKING = 'False'
    INVENTORY = 'ansible/inventory.ini'
    PLAYBOOK  = 'ansible/deploy.yml'
    ARTIFACT_VERSION = readFile('version').trim()
    EMAIL_USERNAME = 'Jenkins'
    JENKINS_HOME = '/var/lib/jenkins'
  }

  stages {
    stage('Tests') {
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
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    // stage('Create Docker image') {
    //   when { branch 'master' }
    //   steps {
    //       sh "docker build -t cicd-app:${ARTIFACT_VERSION} ."
    //   }
    // }

    stage('Approval for Production') {
      steps {
        timeout(time: 1, unit: 'HOURS') {
          input message: 'Deploy to production?', ok: 'Approve'
        }
      }
    }

    stage('Promote dev image to production-ready (remove dev suffix)') {
      steps {
        sh '''
        docker tag viyd/cicd-app:${ARTIFACT_VERSION}-dev viyd/cicd-app:${ARTIFACT_VERSION}
        '''
      }
    }

    stage('Push Docker image to Docker Hub') {
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
      steps {
        withCredentials([string(credentialsId: 'ansible-vault-pass', variable: 'VAULTPASS')]) {

          sh '''
            KCFG=$(mktemp)
            chmod 600 "$KCFG"

            trap "rm -f $KCFG" EXIT

            ansible-vault view ${JENKINS_HOME}/kubeconfig.vault \
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
