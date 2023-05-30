pipeline {
  agent any
  environment {
    DOCKER_REGISTRY_USERNAME = credentials('DOCKER_REGISTRY_USERNAME')
    DOCKER_REGISTRY_PASSWORD = credentials('DOCKER_REGISTRY_PASSWORD')
  }

  // Stage ví dụ để nhánh bugfix và feature sẽ thực thi quá trình test ở CI.
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

    // Stage build Docker image, chỉ áp dụng cho  2 nhánh develop và release
    stage('Build Docker Image') {
      when {
        anyOf {
          branch 'develop'
          tag pattern: "^v(\\d+(?:\\.\\d+)*)\$", comparator: "REGEXP" 
        }
      }
      steps {
        sh '''
          echo "Starting to build docker image"
          docker build -t orezfu/obo:v1.${BUILD_NUMBER} -f Dockerfile .
        '''
      }
    }

   // Với pipeline của nhánh develop, push docker image lên Docker Hub
    stage('Push Docker Image in develop') {
      when {
        branch 'develop'
      }
      // Đánh tag dev cho image
      steps {
        sh '''
          echo "Tag image to dev and push image"
          docker tag orezfu/obo:v1.${BUILD_NUMBER} orezfu/obo:v1.${BUILD_NUMBER}-dev
          echo ${DOCKER_REGISTRY_PASSWORD} | docker login -u ${DOCKER_REGISTRY_USERNAME} --password-stdin
          docker push "orezfu/obo:v1.${BUILD_NUMBER}-dev"
        '''
      }
    }

    // Deploy tới môi trường development, tương ứng là namespace dev trên Kubernetes
    stage('Deploy to development environment') {
      when {
        branch 'develop'
      }
      steps {
        script {
          sh "echo 'Deploy to kubernetes'"
          echo "Update tag in deployment"
          def filename = 'manifests/deployment.yaml'
          def data = readYaml file: filename
          data.spec.template.spec.containers[0].image = "orezfu/obo:v1.${BUILD_NUMBER}-dev"
          sh "rm $filename"
          writeYaml file: filename, data: data
          sh "cat $filename"
          echo "Update service Node Port"
          def servicefile = 'manifests/service.yaml'
          def servicedata = readYaml file: servicefile
          servicedata.spec.ports[0].nodePort = 30190
          sh "rm $servicefile"
          writeYaml file: servicefile, data: servicedata
        }
        withKubeConfig([credentialsId: 'kubeconfig', serverUrl: 'https://10.0.2.15:6443']) {
          sh 'curl -LO "https://storage.googleapis.com/kubernetes-release/release/v1.20.5/bin/linux/amd64/kubectl"'
          sh 'chmod u+x ./kubectl'
          sh './kubectl apply -f manifests -n dev'
        }
      }
    }

    // Nếu là tag, yêu cầu nhập vào version cho ứng dụng để đánh tag và triển khai.
    stage('Push Docker Image in release') {
      when {
        tag pattern: "^v(\\d+(?:\\.\\d+)*)\$", comparator: "REGEXP"
      }
      steps {
        sh '''
          echo "Tag image to releae and push image"
          docker tag <dockerhub_account>/obo:v1.${BUILD_NUMBER} orezfu/obo:${TAG_NAME}
          echo ${DOCKER_REGISTRY_PASSWORD} | docker login -u ${DOCKER_REGISTRY_USERNAME} --password-stdin
          docker push "<dockerhub_account>/obo:${TAG_NAME}"
        '''
      }
    }

    stage('Deploy to release environment') {
      when {
        // beforeInput true
        tag pattern: "^v(\\d+(?:\\.\\d+)*)\$", comparator: "REGEXP"
      }
      // Xac nhan deploy
      //input {
      //  message "Approve to deploy PRD?"
      //  ok "Confirm"
      //}
      steps {
        script {
          sh "echo 'Deploy to kubernetes'"
          echo "Update deployment.yaml"
          def filename = 'manifests/deployment.yaml'
          def data = readYaml file: filename
          data.spec.template.spec.containers[0].image = "orezfu/obo:${TAG_NAME}"
          sh "rm $filename"
          writeYaml file: filename, data: data
          sh "cat $filename"
          echo "Update service Node Port"
          def servicefile = 'manifests/service.yaml'
          def servicedata = readYaml file: servicefile
          servicedata.spec.ports[0].nodePort = 30290
          sh "rm $servicefile"
          writeYaml file: servicefile, data: servicedata
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