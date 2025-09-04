pipeline {
  agent any
  environment {
    REGISTRY = "docker.io"
    REGISTRY_NAMESPACE = "khz0825" // ← 본인 Docker Hub ID
    IMAGE_NAME = "sample-app"
    COMMIT = "${env.GIT_COMMIT?.take(7) ?: 'local'}"
    IMAGE = "${REGISTRY}/${REGISTRY_NAMESPACE}/${IMAGE_NAME}:${COMMIT}"
    KUBE_NAMESPACE = "demo"
  }
  stages {
    stage("Checkout"){ steps{ checkout scm } }
    stage("Docker Login"){
      steps{
        withCredentials([usernamePassword(credentialsId: "DOCKERHUB_CRED", usernameVariable: "U", passwordVariable: "P")]){
          sh 'echo "$P" | docker login -u "$U" --password-stdin'
        }
      }
    }
    stage("Build & Push"){
      steps{
        sh 'docker build -t ${IMAGE} .'
        sh 'docker push ${IMAGE}'
      }
    }
    stage("Prepare Kubeconfig"){
      steps{
        withCredentials([string(credentialsId: "KUBECONFIG_CONTENT", variable: "KCONF")]){
          sh '''
            mkdir -p .kube
            printf "%s" "$KCONF" | tr -d '\\r' > .kube/config
            export KUBECONFIG=$PWD/.kube/config
            kubectl config current-context
          '''
        }
      }
    }
    stage("Deploy"){
      steps{
        sh '''
          export KUBECONFIG=$PWD/.kube/config
          kubectl apply -f k8s/namespace.yaml
          sed "s|__IMAGE__|${IMAGE}|g" k8s/deployment.yaml | kubectl apply -f -
          kubectl apply -f k8s/service.yaml
        '''
      }
    }
    stage("Health"){
      steps{
        sh '''
          export KUBECONFIG=$PWD/.kube/config
          kubectl -n ${KUBE_NAMESPACE} rollout status deploy/sample-app --timeout=120s
          kubectl -n ${KUBE_NAMESPACE} get pods -o wide
          kubectl -n ${KUBE_NAMESPACE} get svc sample-svc -o wide
        '''
      }
    }
  }
  post { always { sh 'docker logout || true' } }
}
