pipeline {
  agent none

  environment {
    // For building Helm client.
    HELM_URL = 'https://storage.googleapis.com/kubernetes-helm'
    HELM_TARBALL = 'helm-v2.2.0-linux-amd64.tar.gz'
    CHART = 'stable/drupal'

    // For setting GitHub deploy status.
    CONTEXT = 'scottrigby/jenkins-helm-github-pr-envs'
    // Note you also need to set the following Kubernetes Pod Template EnvVars
    // in Manage Jenkins > Configure System:
    // - GITHUB_AUTH_TOKEN
    // You can also test locally on minikube by setting faux values for these
    // EnvVars, which are otherwise set by the GitHub Pull Request Builder
    // Plugin: https://wiki.jenkins-ci.org/display/JENKINS/GitHub+pull+request+builder+plugin
    // - ghprbActualCommit
    // - ghprbPullId
    // - ghprbGhRepository
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

          if helm status pr-env-$ghprbPullId > /dev/null 2>&1; then
            helm delete --purge pr-env-$ghprbPullId
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
          helm install --name pr-env-$ghprbPullId $CHART --set Persistence.Enabled=false

          # Create GitHub API status.
          URL="https://api.github.com/repos/$ghprbGhRepository/deployments"
          HEADER="Authorization: token $GITHUB_AUTH_TOKEN"

          # Write JSON to file because variable expansion in JSON in
          # pipeline DSL is challenging. We then cURL from file below
          # using @.
          cat << EOF > DATA.txt
{
  "ref": "$ghprbActualCommit",
  "environment": "PR env",
  "auto_merge": false,
  "context": "$CONTEXT"
}
EOF
          # Create GitHub deploy notification, and store response.
          RESPONSE=$(curl $URL -H "$HEADER" -d @DATA.txt)

          # Save status ID for next step.
          # Use jq library until there is a better way to parse the value from
          # JSON in shell.
          wget https://github.com/stedolan/jq/releases/download/jq-1.5/jq-linux64
          chmod +x jq-linux64
          DEPLOY_ID=$(echo $RESPONSE | ./jq-linux64 '.id')
          echo "$DEPLOY_ID" > DEPLOY_ID.txt
        '''

        // For now, set deploy status in the same pipeline stage, because with
        // the k8s plugin some jobs (when enough time lapses between stages) do
        // execute in a new pod, so files can not be stored in the same
        // workspace to pass data between stages. We would need this to get the
        // deploy ID to set it's status.
        // Do this in a separate shell step though, so there is enough of a
        // delay for the PR env pod to be ready.
        sh '''
          DEPLOY_ID=$(cat DEPLOY_ID.txt)

          curl -O $HELM_URL/$HELM_TARBALL
          tar xzfv $HELM_TARBALL -C /home/jenkins && rm $HELM_TARBALL
          PATH=/home/jenkins/linux-amd64/:$PATH
          helm init --client-only

          # Update the GitHub deploy status accordingly.
          HEADER="Authorization: token $GITHUB_AUTH_TOKEN"
          DEPLOY_ID=$(cat DEPLOY_ID.txt)
          URL="https://api.github.com/repos/$ghprbGhRepository/deployments/$DEPLOY_ID/statuses"
          if helm list --deployed --short pr-env-$ghprbPullId > /dev/null 2>&1; then
            cat << EOF > DATA.txt
{
  "state": "success",
  "description": "PR deployment succeeded",
  "target_url": "https://$ghprbPullId.jenkins-helm-github-pr-envs.com"
}
EOF
            curl $URL -H "$HEADER" -d @DATA.txt
          else
            DATA='{"state": "error"}'
            curl $URL -H "$HEADER" -d "$DATA"
            # If there was a deploy problem, also fail Jenkins job.
            exit 1
          fi
        '''
      }
    }
  }
}
