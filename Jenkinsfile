pipeline {
  agent none

  environment {
    HELM_URL = 'https://storage.googleapis.com/kubernetes-helm'
    HELM_TARBALL = 'helm-v2.2.0-linux-amd64.tar.gz'
    CHART = 'stable/drupal'
    POD = 'drupal'
  }

  stages {
    stage('Clear any stale PR env') {
      agent any

      steps {
        sh '''
          curl -O $HELM_URL/$HELM_TARBALL
          tar xzfv $HELM_TARBALL -C /home/jenkins && rm $HELM_TARBALL
          PATH=/home/jenkins/linux-amd64/:$PATH
          helm init --client-only

          if helm status pr-env > /dev/null 2>&1; then
            helm delete --purge pr-env
            echo 'Deleted PR env.'
          else
            echo 'No PR env to delete.'
          fi
        '''
      }
    }

    stage('Build PR env') {
      agent any

      steps {
        sh '''
          curl -O $HELM_URL/$HELM_TARBALL
          tar xzfv $HELM_TARBALL -C /home/jenkins && rm $HELM_TARBALL
          PATH=/home/jenkins/linux-amd64/:$PATH
          helm init --client-only

          # Minikube requires persistence to be disabled.
          # @todo Parameterize persistence for local vs non-local.
          helm install --name pr-env $CHART --set Persistence.Enabled=false
        '''
      }
    }
    stage('Test PR env') {
      agent any

      steps {
        echo 'To-do add tests'
      }
    }
  }
}
