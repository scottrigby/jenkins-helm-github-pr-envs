Jenkins Helm GitHub PR envs
===========================

Why?
----
Because it's awesome!

A Jenkins pipeline DSL that creates GitHub PR environments using Kubernetes and
 Helm, but is otherwise dependency-free.

Installation
------------
1. Install [stable/jenkins kubernetes chart](https://github.com/kubernetes/charts/tree/master/stable/jenkins).
    - See `Demo chart install details` section below.
1. Install additional Jenkins plugins:
    - [GitHub Integration plugin](https://plugins.jenkins.io/github-pullrequest),
      a well-maintained, rewrite of ghprb-plugin that works with Jenkins pipeline.
      See [history](https://github.com/KostyaSha/github-integration-plugin/blob/master/docs/History.adoc).
      Alternately we could use the [GitHub Organization Folder plugin](https://plugins.jenkins.io/github-organization-folder),
      which includes PR functionality, and may be more robust in the long run.

Configuration
-------------
1. Configure [Git plugin](https://plugins.jenkins.io/git)
    - Under `Jenkins > Manage Jenkins > Configure System > GitHub`, Add a GitHub
      server. For credentials I use `secret text`, but there are several options.
      If you want to use GitHub webhooks, also check the `Manage hooks` box.
1. Under `Jenkins > Manage Jenkins > Configure System > Images > Kubernetes Pod Template > EnvVars`,
   click `Add Environment Variable`
    - Set `Key` to `GITHUB_AUTH_TOKEN`, and `Value` to a GitHub Auth Token (can
      be the same token as `secret text` above if the token's user has admin
      access to the repo).
    - In future this extra step should be unnecessary.
1. Create a new Pipeline job
1. Under `General > GitHub project > Project url` add your GitHub project URL.
1. Under `Build Triggers` check `GitHub Pull Requests`.
    - For `Trigger Mode` choose a `Hooks` option if you want to use GitHub
      webhooks (or choose `Cron` if you want to poll, for example when testing
      locally).
    - For `Trigger Events` add `Pull Request Opened`, and `Commit changed`.
    - For more help see the [plugin docs](https://github.com/KostyaSha/github-integration-plugin/blob/master/docs/Configuration.adoc#pull-requests-trigger).
1. Under `Pipeline > Definition` select `Pipeline script from SCM`.
    - For `SCM` select `Git`.
    - For `Repositories > Repository URL` point to this repo (or your fork, etc).
    - You will need to add Credentials here again. I use `SSH Username with
      private key` (note you can not use `secret text` here).
    - For more help see [Jenkins doc page](https://jenkins.io/pipeline/getting-started-pipelines/#loading-pipeline-scripts-from-scm)

Usage
-----
1. Create a PR in your GitHub project:
    - A Helm PR environment should be created, and the GitHub PR should be
      notified.
1. Push changes to the PR branch:
    - The PR env should rebuild, and notify GitHub PR.
1. Close the PR:
    - The PR env should be deleted, and notify GitHub PR.

Support
-------
This is meant as an example, not for production use, but if you have problems or
 suggested improvements, opening issues [in this repo](https://github.com/scottrigby/jenkins-helm-github-pr-envs/issues)
 will help make it better for everyone.

Demo chart install details
--------------------------
1. [Install kubectl](https://kubernetes.io/docs/tasks/kubectl/install/)
1. [Install Minikube](https://github.com/kubernetes/minikube)
    - This demo for MacOS also assumes [installing the xhyve driver](https://github.com/kubernetes/minikube/blob/master/DRIVERS.md#xhyve-driver).
      If you choose a different [virtualization option](https://github.com/kubernetes/minikube#requirements)
      you can drop the `--vm-driver` argument in the next step.
1. Start Minikube:

    ```sh
    # See above step notes if you did not install xhyve driver.
    minikube start --vm-driver=xhyve
    ```
1. Install tiller on the Minikube cluster:

    ```sh
    helm init
    ```
1. Install an Ingress controller. The Minikube Ingress controller add-on handles
   this automatically:

    ```sh
    minikube addons enable ingress
    ```
1. Install Jenkins chart:

    ```sh
    # Change to your suit your local hostname pattern preference.
    HOSTNAME='jenkins-demo.local'
    DEMO_NAME='jenkins-demo'
    helm install stable/jenkins \
      # To use Ingress you must also set an Ingress class matching the running
      # Ingress Controller. This class matches the Minikube Ingress controller
      # add-on we enabled in the last step.
      --set Ingress.Annotations.kubernetes.io/ingress.class=nginx \
      --set Master.HostName=$HOSTNAME \
      # Set a name for demo ease.
      --name $DEMO_NAME
    ```
1. Add kubernetes cluser IP and Ingress domain to your hosts file:

    ```sh
    echo "$(minikube ip) $HOSTNAME" | sudo tee -a /etc/hosts
    ```
1. Log into Jenkins at your HostName as "admin" user:

    ```sh
    open http://$HOSTNAME
    ```

    The password is in a kubernetes Secret. You can copy that via the kubernetes
    dashboard:

    ```sh
    open "$(minikube dashboard --url)/#/secret/default/$DEMO_NAME-jenkins"
    ```
1. Finish `Installation`, `Configuration` and `Usage` sections above.
1. After demo, cleanup:

    ```sh
    # Delete the chart from the Minikube cluster.
    helm delete $DEMO_NAME --purge
    # Remove the hostname we added to your hosts file.
    sudo sed -ie "\|$HOSTNAME\$|d" /etc/hosts
    ```
