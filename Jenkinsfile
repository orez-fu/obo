pipeline {
  agent any
  environment {
    DOCKER_REGISTRY_USERNAME = credentials('DOCKER_REGISTRY_USERNAME')
    DOCKER_REGISTRY_PASSWORD = credentials('DOCKER_REGISTRY_PASSWORD')
  }
  parameters {
    string(name: 'IMAGE_VERSION', defaultValue: 'v0.0.0')
  }

  stages {
    stage('Install dependencies and Test') {
      steps {
        echo "pseudo install dependencies"
        sh '''
          echo "pseudo run test case"
          echo "pseudo run integration test"
          echo "pseudo check code quality"
        '''
      }
    }

    stage('Build Docker Image') {
      when {
        anyOf {
          branch 'develop'
          branch 'release'
        }
      }
      steps {
        sh '''
          echo "Starting to build docker image"
          docker build -t orezfu/obo:v1.${BUILD_NUMBER} -f Dockerfile .
        '''
      }
    }

    stage('Push Docker Image in develop') {
      when {
        branch 'develop'
      }
      steps {
        sh '''
          echo "Tag image to dev and push image"
          docker tag orezfu/obo:v1.${BUILD_NUMBER} orezfu/obo:v1.${BUILD_NUMBER}-dev
          echo ${DOCKER_REGISTRY_PASSWORD} | docker login -u ${DOCKER_REGISTRY_USERNAME} --password-stdin
          docker push "orezfu/obo:v1.${BUILD_NUMBER}-dev"
        '''
      }
    }

    stage('Push Docker Image in release') {
      when {
        branch 'release'
      }
      input {
        message "Enter release version... (example: v1.2.3)"
        ok "Confirm"
        parameters {
          string(name: "IMAGE_TAG", defaultValue: "v0.0.0")
        }
      }
      steps {
        sh '''
          echo "Tag image to releae and push image"
          docker tag orezfu/obo:v1.${BUILD_NUMBER} orezfu/obo:${IMAGE_TAG}
          echo ${DOCKER_REGISTRY_PASSWORD} | docker login -u ${DOCKER_REGISTRY_USERNAME} --password-stdin
          docker push "orezfu/obo:${IMAGE_TAG}"
        '''
        sh '''
          echo "Set variable to use in next stage"
          env.IMAGE_VERSION = ${IMAGE_TAG}
        '''
      }
    }

    stage('Deploy to development environment') {
      when {
        branch 'develop'
      }
      steps {
        script {
          sh "echo 'Deploy to kubernetes'"
          def filename = 'manifests/deployment.yaml'
          def data = readYaml file: filename
          data.spec.template.spec.containers[0].image = "orezfu/obo:v1.${BUILD_NUMBER}-dev"
          sh "rm $filename"
          writeYaml file: filename, data: data
          sh "cat $filename"
        }
        withKubeConfig([credentialsId: 'kubeconfig', serverUrl: 'https://10.0.2.15:6443']) {
          sh 'curl -LO "https://storage.googleapis.com/kubernetes-release/release/v1.20.5/bin/linux/amd64/kubectl"'  
          sh 'chmod u+x ./kubectl'  
          sh './kubectl apply -f manifests -n dev'
        }
      }
    }

    stage('Deploy to release environment') {
      when {
        branch 'release'
      }
      steps {
        script {
          sh "echo 'Deploy to kubernetes'"
          def filename = 'manifests/deployment.yaml'
          def data = readYaml file: filename
          data.spec.template.spec.containers[0].image = "orezfu/obo:${env.IMAGE_TAG}"
          sh "rm $filename"
          writeYaml file: filename, data: data
          sh "cat $filename"
        }
        withKubeConfig([credentialsId: 'kubeconfig', serverUrl: 'https://10.0.2.15:6443']) {
          sh 'curl -LO "https://storage.googleapis.com/kubernetes-release/release/v1.20.5/bin/linux/amd64/kubectl"'  
          sh 'chmod u+x ./kubectl'  
          sh './kubectl apply -f manifests -n release'
        }
      }
    }
  }
}
