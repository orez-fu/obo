pipeline {
  agent any
  environment {
    DOCKER_REGISTRY_USERNAME = credentials('DOCKER_REGISTRY_USERNAME')
    DOCKER_REGISTRY_PASSWORD = credentials('DOCKER_REGISTRY_PASSWORD')
  }

  stages {
    stage('Build') {
      steps {
        sh '''
          echo "Starting to build docker image"
          docker build -t orezfu/obo:v1.${BUILD_NUMBER} -f Dockerfile .
        '''
        sh '''
          echo "Starting to push docker image"
          echo ${DOCKER_REGISTRY_PASSWORD} | docker login -u ${DOCKER_REGISTRY_USERNAME} --password-stdin
          docker push "orezfu/obo:v1.${BUILD_NUMBER}"
        '''
      }
    }
    stage('Deploy') {
      steps {
        withCredentials([gitUsernamePassword(credentialsId: 'jenkins_github_pac', gitToolName: 'Default')]) {
          sh 'rm -rf obo-manifest'
          sh 'git clone https://github.com/orez-fu/obo-manifest.git'
          sh 'cd obo-manifest'
        }
        script {
          sh "echo 'Update deployment manifest'"
          def filename = 'obo-manifest/dev/deployment.yaml'
          def data = readYaml file: filename
          data.spec.template.spec.containers[0].image = "orezfu/obo:v1.${BUILD_NUMBER}"
          sh "rm $filename"
          writeYaml file: filename, data: data
          sh "cat $filename"
        }
        withCredentials([gitUsernamePassword(credentialsId: 'jenkins_github_pac', gitToolName: 'Default')]) {
          sh '''
            cd obo-manifest
            git config user.email "jenkins@example.com"
            git config user.name "Jenkins"
            git add dev/deployment.yaml
            git commit -am "update image to tag v1.${BUILD_NUMBER}"
            git push origin master
          '''
        }
      }
    }
  }
}
