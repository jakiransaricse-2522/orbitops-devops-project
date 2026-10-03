pipeline {
  agent any
  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }
  parameters {
    booleanParam(name: 'DEPLOY', defaultValue: true, description: 'Deploy the successfully built image to the local OrbitOps lab')
  }
  environment {
    IMAGE_NAME = 'orbitops-web'
    IMAGE_TAG = "build-${BUILD_NUMBER}"
    APP_CONTAINER = 'orbitops_web'
    APP_PORT = '8080'
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Install dependencies') {
      steps { sh 'npm ci' }
    }
    stage('Validate and build') {
      steps {
        sh 'npm run build'
      }
    }
    stage('Build Docker image') {
      steps {
        sh 'docker build --pull -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest .'
      }
    }
    stage('Deploy with Ansible') {
      when { expression { return params.DEPLOY } }
      steps {
        sh '''
          ansible-playbook -i ansible/inventory.ini ansible/deploy.yml \\
            -e "docker_image=${IMAGE_NAME}:${IMAGE_TAG}" \\
            -e "app_port=${APP_PORT}"
        '''
      }
    }
    stage('Verify deployment') {
      when { expression { return params.DEPLOY } }
      steps {
        sh '''
          for attempt in $(seq 1 20); do
            if curl --fail --silent http://host.docker.internal:${APP_PORT}/ >/dev/null; then
              echo "OrbitOps is responding on port ${APP_PORT}"
              exit 0
            fi
            sleep 3
          done
          echo "Application health check failed"
          exit 1
        '''
      }
    }
  }
  post {
    success { echo 'OrbitOps pipeline completed successfully.' }
    failure { echo 'Pipeline failed. Review the stage logs above.' }
    always { cleanWs() }
  }
}
