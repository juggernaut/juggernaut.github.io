---
layout: post
title:  "Kubernetes: Building a CI/CD pipeline using ArgoCD, Tekton and Kaniko"
date:   2021-11-04 14:10:32
categories: technology kubernetes ci/cd gitops
---

In a [previous post](https://www.ameyalokare.com/technology/kubernetes/ci/cd/github-actions/2021/10/30/exploring-kubernetes-automated-deployment-part-1.html), I created a simple deployment pipeline using Github Actions. A major drawback was having to trigger deployments from outside the Kubernetes cluster which risks exposing credentials. Additionally, a push-based approach means that a transient error when invoking the deployment operation would fail the pipeline and require manual intervention.

Kubernetes-native CI/CD tools promise to solve these problems. The idea is to run operators in Kubernetes that will run the pipeline within the cluster based on changes in your code repo.

I will re-use the [sample app](https://github.com/juggernaut/k8s-demo-emp-api) (an `Employees` API) that I created for my [previous blog post](https://www.ameyalokare.com/technology/kubernetes/2021/11/04/exploring-kubernetes-deploying-multiple-environments.html).

## ArgoCD

My plan was to start with the CD portion first, keeping the CI portion in Github Actions. After some cursory research, ArgoCD seemed to be the clear winner here, so I chose it. 

[Installation](https://argo-cd.readthedocs.io/en/stable/getting_started/#1-install-argo-cd) was straightforward. However, to expose it using an ingress rule, I had to pass "--enable-ssl-passthrough" to my ingress-nginx controller. On DigitalOcean Kubernetes, this option is not set by default, so I needed to patch the controller deployment:

{% highlight bash %}
$ kubectl -n ingress-nginx patch deployment ingress-nginx-controller --type=json -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--enable-ssl-passthrough"}]'
{% endhighlight %}

To make it easier to access from my local machine, I added a `/etc/hosts` entry with a fake hostname (argocd.empapi.io) pointing to the public IP of the ingress load balancer. Then, I created an ingress rule (note the annotations for ssl passthrough):

{% highlight yaml %}
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  rules:
  - host: argocd.empapi.io
    http:
      paths:
      - backend:
          serviceName: argocd-server
          servicePort: https
{% endhighlight %}

Manifests repo

We'll keep our manifests in a separate git repo and structure it in the following way:

```
manifests
├── base
│   ├── emp-api-deployment.yaml
│   ├── emp-api-ingress.yaml
│   ├── emp-api-svc.yaml
│   └── kustomization.yaml
├── production
│   ├── increase_replicas.yaml
│   ├── ingress_hostname_patch.yaml
│   └── kustomization.yaml
└── staging
    ├── ingress_hostname_patch.yaml
    └── kustomization.yaml
```

This is the recommended layout for gitops with kustomize - base configs with overlays for different environments.
You can view the manifests for the sample app on [GitHub](https://github.com/juggernaut/k8s-demo-emp-api-manifests/tree/main/manifests)

What ArgoCD does is conceptually very simple - it takes a repo with a set of manifests and makes sure they are synced with the cluster. You can set it to continually watch a repo for changes, or manually trigger the sync process. It supports many kinds of manifests, including kustomize which is perfect for us.

The basic resource type is an "Application". Let's create an ArgoCD app for the staging environment:

{% highlight bash %}
argocd app create k8s-demo-emp-api-staging --repo https://github.com/juggernaut/k8s-demo-emp-api-manifests --path manifests/staging --dest-server https://kubernetes.default.svc --dest-namespace staging
{% endhighlight %}

The command is fairly self-explanatory. Note that the destination server is the local kubernetes API server. ArgoCD can deploy to different clusters which is probably what you want in an actual production setup -- separate clusters for dev/staging/prod and ArgoCD running in an "admin" cluster capable of deploying to each.

When you first create the app, the resources in the manifest are in state "OutOfSync":

{% highlight bash %}
$ argocd app get k8s-demo-emp-api-staging
.....
.....
GROUP              KIND        NAMESPACE  NAME                         STATUS     HEALTH   HOOK  MESSAGE
                   Service     staging    k8s-emp-api-svc              OutOfSync  Healthy
apps               Deployment  staging    k8s-demo-emp-api-deployment  OutOfSync  Healthy
networking.k8s.io  Ingress     staging    ingress-to-emp-api           OutOfSync  Healthy
{% endhighlight %}

Let's sync it:

{% highlight bash %}
$ argocd app sync k8s-demo-emp-api-staging
{% endhighlight %}

Then, wait for it to complete:

{% highlight bash}
$ argocd app wait k8s-demo-emp-api-staging
....
....
GROUP              KIND        NAMESPACE   NAME                         STATUS  HEALTH   HOOK  MESSAGE
                   Service     staging  k8s-emp-api-svc                 Synced  Healthy        service/k8s-emp-api-svc configured
apps               Deployment  staging  k8s-demo-emp-api-deployment     Synced  Healthy        deployment.apps/k8s-demo-emp-api-deployment configured
networking.k8s.io  Ingress     staging  ingress-to-emp-api              Synced  Healthy        ingress.networking.k8s.io/ingress-to-emp-api configured
{% endhighlight %}

You can verify that the deploy was successful by checking the deployment status as well.

## Tekton

ArgoCD, as the name suggests, only does the deployment portion of a proper CI/CD pieline. We'll need to pair it up with another tool that will do the CI portion and hook into ArgoCD to do deployment. I was planning to use Jenkins since I have experience with it, but coincidentally, I came across the DigitalOcean challenge. The challenge requires you to use Tekton. I'd never heard of Tekton before, and their claim of "K8s native CI/CD tool" got me interested. 

Tekton is "serverless" in that you don't need to maintain a Jenkins-like server. Everything is run in containers on Kubernetes. Pipelines and tasks are specified as CRDs so you can just `kubectl apply` them. Keep in mind that the learning curve is a bit steep and there's a lot of [concepts](https://tekton.dev/docs/concepts/) to learn. I recommend going through the official docs, but here's a summary:

* A _step_ is some command(s) that run in a container
* A _task_ is a series of steps in order. A task runs in a K8s pod. You can write your own tasks but there's also a collection of reusable tasks in the [Tekton Catalog](https://github.com/tektoncd/catalog)
* A _pipeline_ is a DAG of tasks. Pipelines are what you'd normally trigger when events of interest occur (a git push for example)
* An _eventListener_ is a K8s service that listens for events from external systems e.g a webhook from Github.
* A _taskRun_ is a specific execution of a task
* A _pipelineRun_ is a specific execution of a pipeline. You'll want to inspect the _pipelineRun_ to debug issues. 

## Test PR pipeline

We'll create two pipelines; a pipeline to test PRs, and a post-merge deployment pipeline. Let's start with a barebones version of the the "test PR" pipeline, which simply checks out a git repo and merge a PR branch locally. First, we'll need to install the `git-cli` task from the Tekton catalog:

{% highlight bash %}
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-cli/0.3/git-cli.yaml
{% endhighlight %}

The pipeline YAML that I've annotated with comments:

{% highlight yaml %}
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: merge-pr-pipeline
spec:
  # Declare workspaces required by the pipeline. Workspaces are mounted as volumes inside pods running _steps_.
  workspaces:
    # The main build workspace. An actual K8s PVC needs to be bound to it when executing the pipeline
    - name: merge-pr-ws
    # Contains the SSH private key for accessing the git repo. It will be bound to a K8s secret at runtime
    - name: git-ssh-creds
  # Values for these params will eventually come from Github webhooks.
  params:
    - name: prnumber
      description: The PR number
    - name: prbranch
      description: The PR branch name
   tasks:
    - name: clone-and-merge-pr
      taskRef:
        name: git-cli
      # Associate workspaces declared by the `git-cli` task with workspaces from our pipeline.
      workspaces:
        # The .ssh directory containing SSH credentials to the git repo
        - name: ssh-directory
          workspace: git-ssh-creds
        # Where the source will be checked out
        - name: source
          workspace: merge-pr-ws
      params:
        - name: GIT_USER_NAME
          value: "build-bot"
        - name: GIT_USER_EMAIL
          value: "build-bot@empapi.io"
        - name: GIT_SCRIPT
          value: |
            git clone git@github.com:juggernaut/k8s-demo-emp-api.git
            cd k8s-demo-emp-api
            git checkout -b merge-pr-$(params.prnumber)
            git merge --no-ff origin/$(params.prbranch) 
{% endhighlight %}

I generated an SSH key-pair using `ssh-keygen` and added the public key as a [deploy key](https://docs.github.com/en/developers/overview/managing-deploy-keys) to my repo. Then, I created a directory `git-ssh-creds` with the private key inside it and created a secret out of it:

```
kubectl create secret generic git-ssh-creds --from-file=git-ssh-creds
```

Apply the above yaml using `kubectl apply -f`. Let's create a `PipelineRun` resource that will enable us to actually run the pipeline:

{% highlight yaml %}
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: merge-pr-pipeline-run-1
spec:
  pipelineRef:
    name: merge-pr-pipeline
  params:
    - name: prnumber
      value: "1"
    - name: prbranch
      value: juggernaut-patch-1
  workspaces:
    - name: git-ssh-creds
      secret:
        secretName: git-ssh-creds
    - name: merge-pr-ws # this workspace name must be declared in the Pipeline
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce # access mode may affect how you can use this volume in parallel tasks
          resources:
            requests:
              storage: 1Gi # Note that DigitalOcean doesn't allow anything under 1Gi
{% endhighlight %}

To test the pipeline, I created a dummy PR on the repo `juggernaut-patch-1`. As mentioned earlier, we bind the SSH credentials secret to the `git-ssh-creds` workspace. For the build workspace, we'll request a K8s Persistent Volume Claim (PVC) with the specified storage. The storage will be dynamically provisioned at runtime. Applying the pipeline run resource will kick off pipeline execution. You can tail logs from the pipeline run:

{% highlight bash %}
$ tkn pipelinerun logs --last -f
{% endhighlight %}

NOTE: You will need to manually delete the pipeline run to clean up the associated PVC that is created dynamically. Use `tkn pipelinerun delete --all` to clean up all finished pipeline runs. You can also specify a pre-provisioned static PV instead to avoid dynamic provisioning.

Next, we'll verify that the PR is good by running tests. Let's create a custom task to run golang tests. There already exists a catalog task to do this, but we need one that will output a "succeded" or "failed" result that we can pass back to GitHub to set the commit status. The catalog task simply returns a non-zero status and fails the pipeline if the tests fail. Again, I've provided inline comments where required:

{% highlight yaml %}
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: golang-test-write-result
spec:
  params:
  - name: subdir
    description: subdir in the workspace containing the golang source
    default: ./
  - name: packages
    description: packages to test
    default: ./...
  results:
  # Results can be passed from one task to another in the same pipeline
  - name: go-test-result
    description: >-
     test result; can be either 'succeeded' or 'failed'
  workspaces:
  - name: source
  steps:
    - name: run-go-test
      image: golang:1.17
      onError: continue # make sure that we don't abort the pipeline if tests fail
      workingDir: "$(workspaces.source.path)/$(params.subdir)"
      script: go test -v $(packages)
    - name: write-test-result
      image: alpine
      script: |
        # Access the previous step's exit code with `$(steps.step-<stepname>.exitCode.path)`
        if [ `cat $(steps.step-run-go-test.exitCode.path)` = "0" ]; then
          RESULT=success
        else
          RESULT=failure
        fi
        # Results are written to $(results.<resultname>.path)
        echo -n "$RESULT" | tee $(results.go-test-result.path)
{% endhighlight %}

We can now include this task in our pipeline. Notice the `runAfter` directive to make sure we run the tests after checking out the repo - otherwise, tekton is free to run the tasks in any order (or in parallel).

{% highlight yaml %}
    - name: test-emp-api
      taskRef:
        name: golang-test-write-result
      runAfter:
        - clone-and-merge-pr
      workspaces:
        - name: source
          workspace: merge-pr-ws
      params:
        - name: subdir
          value: k8s-demo-emp-api
        - name: packages
          value: ./api
{% endhighlight %}

I've also added tasks to set Github commit status based on test success/failure, which I'll omit from this post for brevity. You can view the full pipeline YAML [here](https://github.com/juggernaut/k8s-demo-emp-api-manifests/blob/bcf204fcbb876b0cffff5e332121bfed6f2681f8/pipelines/merge-pr-pipeline.yaml)

### Hooking up the pipeline to GitHub

So far, we've only manually triggered the pipeline. For it to be actually useful, we'll need to hook it up to listen to GitHub webhooks. The pipeline should be run whenever a PR is created or updated. `EventListener`s will listen to external webhooks and associated `Trigger`s will kick off pipeline execution. First, we'll create a dedicated service account to run the pipeline, instead of running it as admin:

{% highlight bash %}
$ kubectl create serviceaccount build-bot
{% endhighlight %}

Next, specify the `EventListener`, `TriggerTemplate` and `TriggerBinding`:

{% highlight yaml %}
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerTemplate
metadata:
  name: merge-pr-trigger-template
spec:
  params:
    - name: prnumber
      description: The PR number of the triggering pull request
    - name: prbranch
      description: The PR branch of the triggering pull request
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: PipelineRun
      metadata:
        generateName: merge-pr-pipeline-run-
      spec:
        serviceAccountName: build-bot
        pipelineRef:
          name: merge-pr-pipeline
        params:
          - name: prnumber
            value: $(tt.params.prnumber)
          - name: prbranch
            value: $(tt.params.prbranch)
        workspaces:
          - name: git-ssh-creds
            secret:
              secretName: git-ssh-creds
          - name: merge-pr-ws # this workspace name must be declared in the Pipeline
            volumeClaimTemplate:
              spec:
                accessModes:
                  - ReadWriteOnce # access mode may affect how you can use this volume in parallel tasks
                resources:
                  requests:
                    storage: 1Gi
---
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerBinding
metadata:
  name: merge-pr-trigger-template-binding
spec:
  params:
    - name: prnumber
      value: $(body.number)
    - name: prbranch
      value: $(body.pull_request.head.ref)
---
apiVersion: triggers.tekton.dev/v1alpha1
kind: EventListener
metadata:
  name: pr-listener
spec:
  serviceAccountName: build-bot
  triggers:
    - name: github-pr-listener
      interceptors:
        - name: github-interceptor
          ref:
            name: "github"
            kind: ClusterInterceptor
            apiVersion: triggers.tekton.dev
          params:
          - name: "eventTypes"
            value: ["pull_request"]
      bindings:
      - ref: merge-pr-trigger-template-binding
      template:
        ref: merge-pr-trigger-template
{% endhighlight %}

The idea is that EventListener receives webhooks and passes it to a referenced `TriggerBinding` that can extract values from the webhook body and bind them to parameters. These parameters are accessible in the `TriggerTemplate` that in turn passes them to the pipeline run (yes, this seems quite convoluted!). 

After you apply the above resources, you'll see a k8s service running for the event listener. Let's get the name of the service:

{% highlight bash %}
$ kubectl get el pr-listener -o=jsonpath='{.status.configuration.generatedName}'
{% endhighlight %}

The generated name of the service in this case is `el-pr-listener`. We need to expose this service externally for GitHub to be able to access it. I have an nginx ingress already set up [TODO: talk about other ways], so I created an ingress rule for it:

{% highlight yaml %}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-to-pr-listener
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
        - path: /ghwebhooks(/|$)(.*)
          pathType: Prefix
          backend:
            service:
              name: el-pr-listener
              port:
                number: 8080
{% endhighlight %}

Now, we're ready to create a GitHub webhook pointing to https://<Ingress_Domain>/ghwebhooks. 

### Securing the webhook

We'll need to ensure that we run the pipeline only on requests legitimately originating from GitHub. GitHub allows setting a [secret token](https://docs.github.com/en/developers/webhooks-and-events/webhooks/securing-your-webhooks#setting-your-secret-token) that we can verify on the server-side. Fortunately, Tekton already implements this validation as part of [interceptors](https://tekton.dev/vault/triggers-main/interceptors/#github-interceptors).

Let's generate a secret string:

{% highlight bash %}
$ openssl rand -hex 20
{% endhighlight %}

[Configure this string as a secret](https://docs.github.com/en/developers/webhooks-and-events/webhooks/securing-your-webhooks#setting-your-secret-token) in GitHub. Then, create a K8s secret:

{% highlight bash %}
$ kubectl create secret generic github-pr-webhook-secret --from-literal=secretToken=<generated secret>
{% endhighlight %}

Add the Github interceptor to the event listener:

{% highlight yaml %}
interceptors:
  - name: github-interceptor
    ref:
      name: "github"
      kind: ClusterInterceptor
      apiVersion: triggers.tekton.dev
    params:
    - name: "secretRef"
      value:
        secretName: github-pr-webhook-secret
        secretKey: secretToken
{% endhighlight %}

We still have one more minor issue -- we don't want to run the pipeline when the PR is "closed". Currently, any `pull_request` event will kick off the pipeline. [CEL interceptors](https://tekton.dev/vault/triggers-main/interceptors/#cel-interceptors) help us solve this. Add a filter that only allows "opened", "reopened" and "synchronize" PR events:

{% highlight yaml %}
- name: "CEL filter: only when PRs are opened/reopened/synchronized"
  ref:
    name: "cel"
  params:
  - name: "filter"
    value: "body.action in ['opened', 'reopened', 'synchronize']"
{% endhighlight %}

Find the full YAML for EventListener, Tirgger and TriggerTemplate [here](https://github.com/juggernaut/k8s-demo-emp-api-manifests/blob/bcf204fcbb876b0cffff5e332121bfed6f2681f8/pipelines/triggers-merge-pr-pipeline.yaml).

## CD pipeline

Let's now focus on building the continuous deployment pipeline after the PR is merged [TODO: subscript saying that it can be merged after reviews blah blah]. At a high level, it'll do the following steps:
1. On "push" event from branch "main", start the pipeline
2. Check out repo
3. Run unit tests
4. Build container image and push to registry
5. Deploy image in the `staging` environment
6. Run cluster tests
7. Deploy image to `production` environment

### Building the container image
Steps 1-3 are similar to what we already covered earlier, so I'll skip them for brevity [TODO: link to full pipeline]. Building Docker images from within a container environment could cause [security issues](https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/). Since all Tekton steps run in a container, this presents a problem for us. Cue [Kaniko](https://github.com/GoogleContainerTools/kaniko). Kaniko can build images from `Dockerfile`s without needing access to the docker daemon.

Again, we'll make use of the Tekton catalog to reference the [kaniko task](https://github.com/tektoncd/catalog/tree/main/task/kaniko/0.5).

{% highlight yaml %}
- name: build-and-push-image
  taskRef:
    name: kaniko
  runAfter:
    - test-and-build-app
  workspaces:
    - name: source
      workspace: cd-ws
      # Docker credentials for pushing to DockerHub. Read below for how to configure credentials.
    - name: dockerconfig
      workspace: docker-creds
  params:
    - name: IMAGE
      # Image tag is the short version of the Git commit SHA. Helps easily correlate source code and deployments.
      value: "juggernaut/k8s-demo-emp-api:$(params.short-sha)"
    - name: DOCKERFILE
      value: ./kaniko-dockerfile
    - name: CONTEXT
      value: ./k8s-demo-emp-api
{% endhighlight %}

Kaniko can also push images to a registry. It looks within the "dockerconfig" workspace for a docker-style `config.json`. Create a config.json secret with your docker hub credentials:

{% highlight bash %}
$ CREDS=$(echo -n $DOCKER_HUB_USER:$DOCKER_HUB_PASSWORD | base64)
$ cat << EOF > ./config.json
{
	"auths": {
		"https://index.docker.io/v1/": {
			"auth": "$CREDS"
		}
	}
}
EOF
$ kubectl create secret generic docker-config --from-file=config.json
{% endhighlight %}

We'll mount the docker-config secret as a workspace similar to how we did the git credentials.

### Deploying to staging

Now that we have our application image, we'll deploy it to staging and run some integration tests to validate it before promoting it to production. We already have the deployment piece set up with ArgoCD. All we need to do is update the image version in Git and have ArgoCD sync it. Let's run a task to `kustomize edit` to bump the image tag -- unfortunately, the Tekton catalog doesn't have one handy, so I ended up writing one myself:

{% highlight yaml %}
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: kustomize-set-image-tag
spec:
  params:
  - name: context
    description: context relative to the workspace where kustomize should run
    default: ./
  - name: image
    description: name of the image whose tag you want to bump
  - name: tag
    description: tag to set the image to
  workspaces:
  - name: source
  steps:
    - name: set-image-tag
      image: line/kubectl-kustomize:latest
      workingDir: "$(workspaces.source.path)/$(params.context)"
      script: |
        kustomize edit set image "$(params.image)=$(params.image):$(params.tag)"
{% endhighlight %}

And reference it from the pipeline:

{% highlight yaml %}
- name: bump-staging-image-tag
  taskRef:
    name: kustomize-set-image-tag
  runAfter:
    - build-and-push-image
    - clone-manifests-repo
  workspaces:
    - name: source
      workspace: cd-ws
  params:
    - name: image
      value: juggernaut/k8s-demo-emp-api
    - name: context
      value: ./k8s-demo-emp-api-manifests/manifests/staging
    - name: tag
      value: $(params.short-sha)
{% endhighlight %}

Next, we'll commit the change to Git using the `git-cli` task (skipping details here, [TODO: see]).

We're now ready to sync the app using ArgoCD -- there's a minor catch, though. We've so far run the `argocd` commands as the default `admin` user which has all possible permissions. Following the principle of least privilege, we should only give the Tekton pipeline enough permissions to be able to sync the app, and not delete it for example. Additionally, we should avoid providing the admin password to the pipeline.

ArgoCD has a built-in RBAC system we can use for this. Let's create a `syncbot` ArgoCD user:

{% highlight yaml %}
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  # add a bot user with apiKey capability - we don't need to give it the login capability
  #   apiKey - allows generating API keys
  #   login - allows to login using UI
  accounts.syncbot: apiKey
{% endhighlight %}

Create a role that can only get or sync applications, and bind it to the `syncbot` user:

{% highlight yaml %}
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    p, role:syncer, applications, get, */*, allow
    p, role:syncer, applications, sync, */*, allow

    g, syncbot, role:syncer
{% endhighlight %}

As usual, we'll leverage the [argocd-task-sync-and-wait](https://github.com/tektoncd/catalog/tree/main/task/argocd-task-sync-and-wait/0.1) catalog task in our pipeline. The task requires a secret named `argocd-env-secret` containing credentials. Let's generate an API token for the `syncbot` user and create the required secret:

{% highlight bash %}
$ argocd account generate-token --account syncbot --expires-in 30d
$ kubectl create secret generic argocd-env-secret --from-literal=ARGOCD_AUTH_TOKEN=<auth-token> --from-literal=ARGOCD_USERNAME=syncbot
{% endhighlight %}

We also need to pass the ArgoCD server address. Since we are running it on the same cluster, simply give it the [K8s service DNS](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) name:

{% highlight bash %}
$ kubectl create configmap argocd-env-configmap --from-literal=ARGOCD_SERVER=argocd-server.argocd.svc.cluster.local
{% endhighlight %}

Finally, use the task in our pipeline:

That's it! I'll skip describing a couple more steps like running integration tests and promoting the deployment to prod because they're similar to what I've already covered. In any case, you can find the full pipeline configuration here (TODO: link)

## Conclusion

ArgoCD and Tekton are powerful tools that can be combined to build Kubernetes-native CI/CD pipelines. The learning curve (esp. Tekton) and initial setup time are high, but in the end you get a more capable and flexible result than say, Jenkins. Also, the pain of maintaining a Jenkins server goes away. That said, there are downsides too. Writing Tekton pipelines felt similar to programming, but in YAML, and YAML is not a programming language. The experience feels clunky and error-prone, requiring a lot of trial-and-error to get right. To add to that, the documentation is subpar and provided examples use deprecated features like `PipelineResources`.