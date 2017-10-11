pipeline {
  agent none

  environment {
    // For building Helm client.
    // @todo Write a simple Helm Jenkins plugin?
    HELM_URL = 'https://storage.googleapis.com/kubernetes-helm'
    HELM_TARBALL = 'helm-v2.2.0-linux-amd64.tar.gz'
    CHART = 'stable/drupal'

    // For setting GitHub deploy status.
    CONTEXT = 'scottrigby/jenkins-helm-github-pr-envs'
    // Note you also need to set the following Kubernetes Pod Template EnvVars
    // in Manage Jenkins > Configure System:
    // - PR_URL_PATTERN: The PR environment Ingress domain. You will probably
    //   want to add the GITHUB_PR_NUMBER variable somewhere in the domain,
    //   which will be substituted with the dynamic $GITHUB_PR_NUMBER.
    // - GITHUB_AUTH_TOKEN
    // @todo Get auth token from Jenkins credentials somehow. Or maybe it's
    //   better to write a custom GitHub deploy status plugin, if one really can
    //   not be found.
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

          if helm status pr-env-$GITHUB_PR_NUMBER > /dev/null 2>&1; then
            helm delete --purge pr-env-$GITHUB_PR_NUMBER
            echo 'Deleted PR env.'
          else
            echo 'No PR env to delete.'
          fi
        '''
      }
    }

    stage('Build PR env') {
      agent any

      when {
        expression { env.GITHUB_PR_STATE == 'OPEN' }
      }

      steps {
        sh '''
          curl -O $HELM_URL/$HELM_TARBALL
          tar xzfv $HELM_TARBALL -C /home/jenkins && rm $HELM_TARBALL
          PATH=/home/jenkins/linux-amd64/:$PATH
          helm init --client-only

          # Set PR URL.
          PR_URL_PATTERN=${PR_URL_PATTERN:-https://GITHUB_PR_NUMBER.jenkins-helm-github-pr-envs.com}
          TARGET_URL=${PR_URL_PATTERN/GITHUB_PR_NUMBER/$GITHUB_PR_NUMBER}

          # Install the chart.
          helm install --name pr-env-$GITHUB_PR_NUMBER $CHART \
            --set ingress.enabled=true,ingress.hostname="$TARGET_URL"

          # Create GitHub API status.
          # Sadly, this variable is not set by the plugin.
          GITHUB_SOURCE_REPO_NAME="$(echo $GITHUB_PR_URL | awk -F/ '{print $5}')"
          URL="https://api.github.com/repos/$GITHUB_PR_SOURCE_REPO_OWNER/$GITHUB_SOURCE_REPO_NAME/deployments"
          HEADER="Authorization: token $GITHUB_AUTH_TOKEN"

          # Write JSON to file because variable expansion in JSON in
          # pipeline DSL is challenging. We then cURL from file below
          # using @.
          cat << EOF > DATA.txt
{
  "ref": "$GITHUB_PR_HEAD_SHA",
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
          # Sadly, this variable is not set by the plugin.
          GITHUB_SOURCE_REPO_NAME="$(echo $GITHUB_PR_URL | awk -F/ '{print $5}')"
          URL="https://api.github.com/repos/$GITHUB_PR_SOURCE_REPO_OWNER/$GITHUB_SOURCE_REPO_NAME/deployments/$DEPLOY_ID/statuses"
          if helm list --deployed --short pr-env-$GITHUB_PR_NUMBER > /dev/null 2>&1; then
            PR_URL_PATTERN=${PR_URL_PATTERN:-https://GITHUB_PR_NUMBER.jenkins-helm-github-pr-envs.com}
            TARGET_URL=${PR_URL_PATTERN/GITHUB_PR_NUMBER/$GITHUB_PR_NUMBER}
            cat << EOF > DATA.txt
{
  "state": "success",
  "description": "PR deployment succeeded",
  "target_url": "$TARGET_URL"
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
