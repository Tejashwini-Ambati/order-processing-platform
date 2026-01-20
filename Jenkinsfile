pipeline {
  agent any

  environment {
    KUBECONFIG_PATH = "${WORKSPACE}/kubeconfig"
  }

  stages {
    stage('Checkout Code') {
      steps {
        git credentialsId: 'Gitcreds', url: 'https://github.com/Tejashwini-Ambati/order-processing-platform.git', branch: 'main'
      }
    }

    stage('Init Kube Config') {
      steps {
        withCredentials([file(credentialsId: 'kubecreds', variable: 'KUBECONFIG_FILE')]) {
          sh """
            cp $KUBECONFIG_FILE ${KUBECONFIG_PATH}
            export KUBECONFIG=${KUBECONFIG_PATH}
            kubectl version --client=true
          """
        }
      }
    }

    stage('Create Namespaces') {
      steps {
        sh '''
          kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
          kubectl create namespace devopsify --dry-run=client -o yaml | kubectl apply -f -
        '''
      }
    }

    stage('Apply Prometheus RBAC') {
      steps {
        sh 'kubectl apply -f monitoring/prometheus/rbac/ -n monitoring'
      }
    }

    stage('Install Prometheus via Helm') {
      steps {
        sh '''
          helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
          helm repo update
          helm upgrade --install prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
        '''
      }
    }

    stage('Apply Grafana RBAC') {
      steps {
        sh 'kubectl apply -f monitoring/grafana/rbac/ -n monitoring'
      }
    }

    stage('Install Grafana via Helm') {
      steps {
        sh '''
          helm repo add grafana https://grafana.github.io/helm-charts
          helm repo update
          helm upgrade --install grafana grafana/grafana -n monitoring --set adminPassword=admin
        '''
      }
    }

    stage('Apply ServiceMonitors') {
      steps {
        sh '''
          echo "⏳ Waiting for CRDs to be ready..."
          for i in {1..10}; do
            kubectl get crd servicemonitors.monitoring.coreos.com && break || sleep 5
          done

          kubectl apply -f monitoring/servicemonitors/ -n monitoring
        '''
      }
    }

    stage('Build and Push Microservices') {
      steps {
        sh '''
          for service in auth order payment user frontend; do
            docker build -t $DOCKER_REGISTRY/${service}:${IMAGE_TAG} $service
            docker push $DOCKER_REGISTRY/${service}:${IMAGE_TAG}
          done
        '''
      }
    }

    stage('Deploy Microservices via Helm') {
      steps {
        sh '''
          for service in auth order payment user frontend; do
            helm upgrade --install $service helm-charts/$service -n devopsify \
              --set image.repository=$DOCKER_REGISTRY/$service \
              --set image.tag=${IMAGE_TAG}
          done
        '''
      }
    }

    stage('Health Check') {
      steps {
        sh 'kubectl get pods -n devopsify'
      }
    }
  }

  post {
    failure {
      echo '❌ Deployment failed. Please check logs.'
    }
  }
}
