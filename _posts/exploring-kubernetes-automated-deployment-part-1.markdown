---
layout: post
title:  "Exploring Kubernetes: Building an automated deployment pipeline - Part 1"
date:   2021-10-30 18:40:23
categories: technology kubernetes ci/cd github-actions
---

I'd been hearing about this hot technology called "Kubernetes" for years, but never really looked into it in earnest. Now that the madness seems to have died down and Kubernetes is "so 2017", I decided it was time to dive into it.

My previous employer, Twilio, was a pretty early adopter of the cloud. They went all in on AWS in 2008, back when "cloud native" was not a thing. When I joined in 2012, we still hadn't completely figured out the best practices around building software in the cloud -- over the next few years, the engineering org spent a lot of time improving the platform infrastructure around building, deploying and operating our software. As a product engineer with a keen interest in infrastructure, I was a witness to (and occasional contributor) as we built out this tooling.

 Around the same time, several companies were having similar problems and trying to solve them in similar ways. A consensus was emerging around best practices -- immutable infrastructure, blue-green deployments, stateless-as-possible services etc. A lot of these best practices were built into Twilio's custom platform, and I believe, several other companies as well. This meant that a lot of similar work was being done at different places. In the software world, such a situation has almost always resulted in the standardization of a single player who does the "undifferentiated heavy lifting". As I understand it, Kubernetes is this standard. There are benefits to standardization - developers can onboard quicker since they don't have to learn one more deployment and orchestration system. It also gives ops folks skill portability and reduces their support burden, since silly questions from devs can now be "outsourced" to the open source community.

 That said, I wanted to learn for myself how exactly Kubernetes works and whether it delivers on its promises. One problem with the k8s community is that it's full of buzzwords. There are a zillion third party tools to do everything, and I wanted to take more of a "bottom-up" approach using vanilla Kubernetes as much as possible. I learn best by doing, so I fired up a k8s cluster, and did a few toy exercises which I document in this series of blog posts.

## Exercise: Build an automated deployment pipeline
I figured a simple but instructive exercise would be to set up an automated deployment for a Go application. Github Actions is super easy (and free) to get started, so I'm going to use it as the CI/CD platform. I actually followed [Learning Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) and did the exercise of setting up a Kubernetes cluster from scratch using, but I'm not really focused on the admin aspect of k8s here. A managed k8s service is good enough. I toyed with Google Kubernetes Engine (GKE) but couldn't wrap my head around their pricing, so I settled for Digital Ocean which costs $10/month for a minimal cluster.

### Booting the k8s cluster
> **_Prerequisites_**: You'll need to install [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) which allows you to operate on a K8s cluster through the CLI

DO has an intuitive interface to start a k8s cluster. I chose a 1 node control plane, 1 node worker cluster -- we're not really worried about HA here. It took a few minutes to boot. DO provides a "kubeconfig" file which contains the settings to access the cluster. We'll need to copy this file to `~/.kube/config`

Let's make sure we can connect to the cluster:

```bash
$ kubectl cluster-info
Kubernetes master is running at https://d24b5f0b-6900-4caf-9b5c-7739e2c70bbc.k8s.ondigitalocean.com
CoreDNS is running at https://d24b5f0b-6900-4caf-9b5c-7739e2c70bbc.k8s.ondigitalocean.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

All good.

### Sample app
I created a simple Dockerized Go app with a `/demo` endpoint that returns a welcome message. Find it [here](https://github.com/juggernaut/k8s-demo-go-app).

### Manual deployment

Let's try deploying the app manually first. APIs in Kubernetes are declarative - you specify a *deployment resource* and the engine goes and figures out how to make it happen.

Create a `deployment.yaml`:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-demo-go-app-deployment
spec:
  selector:
    matchLabels:
      app: k8s-demo-go-app
  replicas: 1
  template:
    metadata:
      labels:
        app: k8s-demo-go-app
    spec:
      containers:
      - name: k8s-demo-go-app
        image: juggernaut/k8s-demo-go-app:v0.0.1
        ports:
        - containerPort: 8080
```

The important parts are `replicas` which specifies the number of instances of your app you want to run, and `image` which should be the fully qualified docker image name to be pulled.

Let's deploy it:
```bash
$ kubectl apply -f deployment.yaml
```

It takes a second to roll out:
```bash
$ kubectl rollout status deployment/k8s-demo-go-app-deployment
deployment "k8s-demo-go-app-deployment" successfully rolled out
```

Kubernetes has now deployed a single *pod* on our worker node. A *pod* is k8s' unit of deployment -- it *can* house multiple containers, but usually contains just one. 

### Verifying the deployment

The deployed app isn't really accessible publicly - we'll skip doing this for now and access it locally.

Let's get the pod name
```
$ POD_NAME=$(kubectl get pods -l app=k8s-demo-go-app -o jsonpath="{.items[0].metadata.name}")
$ echo $POD_NAME
k8s-demo-go-app-deployment-794594bf85-ln4q7
```

Then port forward `8080` on your local machine to the pod:
```
kubectl port-forward $POD_NAME 8080:8080
```

In another terminal, run a curl to verify it works:
```
$ curl http://localhost:8080/demo
Welcome to the k8s demo app!
```

Sweet.

### Updating the app

Let's go through another iteration of a manual deployment to update the app. Let's change the message returned by the app, and tag the docker image with the next version (v0.0.2). 

```bash
$ docker build . -t juggernaut/k8s-demo-go-app:v0.0.2 && docker push juggernaut/k8s-demo-go-app:v0.0.2
```
Then, we'll edit the `image` version in `deployment.yaml` to v0.0.2 and deploy it:
```bash
$ sed s/v0.0.1/v0.0.2/ deployment.yaml | kubectl apply -f -
```

Once the deploy is complete, check the pod name:
```bash
$ kubectl get pods -l app=k8s-demo-go-app -o jsonpath="{.items[0].metadata.name}"
k8s-demo-go-app-deployment-5c97bfdc6-zgwnr
```

Notice the pod name has changed -  a new pod has been deployed and the old pod has been removed. This is pretty nice, immutable deployments FTW!

### Automating the process

Now, I want to set up a Github Actions pipeline to support the following workflow:
1. I create a release tag on Github
2. Automatically kick off a docker build, tag and push to Docker hub
3. Bump the image tag in the kubernetes deployment resource
4. Apply the deployment

Steps 1\. and 2. is standard docker stuff, so I won't cover it. For steps 3. and 4. I created a separate git repo to house the kubernetes yaml resources. I *could* have put it in the app repo, but tying the app version tightly to the deployment means that you can't easily manually rollback. The idea is to have a separate config repo that defines the state of the cluster. Any update to that repo should trigger a `kubectl apply` to the cluster. This is what the cool kids are calling "GitOps" these days.

### Bumping Docker image version

Inspecting the yaml, I briefly considered unix tools like sed/awk to update the image version, but decided it was too hacky. Unsurprisingly, the k8s community has come up with their own solution to update these yaml files - `kustomize`. It's a declarative way to specify patches on top of the yaml. `kubectl` has native support for applying `kustomize` patches, so you don't need to explicitly modify the original yaml.

Let's create a `kustomization.yaml` in the config repo:
```bash
cat << EOF  > ./kustomization.yaml
resources:
- deployment.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
EOF
```

Let's bump our image version to v0.0.2:

```
kustomize edit set image juggernaut/k8s-demo-go-app=juggernaut/k8s-demo-go-app:v0.0.2
```

This writes a patch spec to `kustomization.yaml`

```bash
$ cat kustomization.yaml
resources:
- deployment.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
images:
- name: juggernaut/k8s-demo-go-app
  newName: juggernaut/k8s-demo-go-app
  newTag: v0.0.2
```

You can check that it produces the correct yaml:
```
$ kustomize build .
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-demo-go-app-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k8s-demo-go-app
  template:
    metadata:
      labels:
        app: k8s-demo-go-app
    spec:
      containers:
      - image: juggernaut/k8s-demo-go-app:v0.0.2
        name: k8s-demo-go-app
        ports:
        - containerPort: 8080
```

Cool, it outputs the updated version. Now, we could replace `deployment.yaml` with this new version and commit it, but a cleaner way is to use `kubectl` directly:

```bash
$ kubectl apply -k .
deployment.apps/k8s-demo-go-app-deployment configured
```

## Setting up the Github Actions workflow

To automate all of the above steps using Github actions, let's start a workflow in `.github/workflows/release.yaml` in the app repo:

```yaml
name: Release CI

on:
  release:
    types: [created]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2
```

This workflow runs when I create a release on Github and currently just checks out the repo. To do "real" continuous delivery, you'll want to hook it up to run on *any* push to master [^1]. 

### Build and tag Docker image
Next, let's build and push the Docker app image. We'll need an access token for Docker Hub which you can create by following the instructions [here](https://docs.docker.com/docker-hub/access-tokens/). Make sure to add the Docker Hub username and access token as [secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets) in Github actions to avoid leaking them.

The release tag is nicely available as `${%raw%}{{ github.event.release.tag_name}}{%endraw%}` in the workflow run, so we'll use it to tag the docker image as well.

```yaml
- name: Login to Docker Hub
  uses: docker/login-action@v1
  with:
    username: ${%raw%}{{ secrets.DOCKER_HUB_USERNAME }}{%endraw%}
    password: ${%raw%}{{ secrets.DOCKER_HUB_ACCESS_TOKEN }}{%endraw%}

- name: Build the k8s demo app Docker image
  run: docker build . --file Dockerfile --tag juggernaut/k8s-demo-go-app:${% raw %}{{ github.event.release.tag_name }}{% endraw %}

- name: Push the k8s demo app Docker image to Docker Hub
  run: docker push juggernaut/k8s-demo-go-app:${%raw%}{{ github.event.release.tag_name }}{%endraw%}
```

### Bump image version
Next, we need to update the image version in the deployment resource using `kustomize`. Remember the yaml resources are in a separate repository, so we'll need to check it out first. Then, we'll create a kustomize patch for the new image version and commit it to the config repo.

> **_NOTE:_** I install `kustomize` using a shell command here to make it explicit, but you should probably use a custom [action](https://github.com/marketplace/actions/setup-kustomize) instead.

```yaml
- name: Checkout k8s config repo
  uses: actions/checkout@v2
  with:
    repository: juggernaut/k8s-demo-config
    token: ${%raw%}{{ secrets.K8S_DEMO_PAT }}{%endraw%}
    path: k8s-demo-config

- name: Install kustomize
  working-directory: k8s-demo-config
  run: |
    curl -s https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh | bash

- name: Update k8s-demo-app image tag
  env:
    RELEASE_TAG: ${%raw%}{{ github.event.release.tag_name }}{%endraw%}
  working-directory: k8s-demo-config
  run: |
    ./kustomize edit set image juggernaut/k8s-demo-go-app=juggernaut/k8s-demo-go-app:$RELEASE_TAG
    git config user.name "k8s demo CI bot"
    git config user.email "<>"
    git add kustomization.yaml
    git commit -m "CI: update image tag to $RELEASE_TAG"
    git push
```

### Deploying the app to k8s

We'll create another workflow to actually deploy the app using `kubectl` in the config repo. This will allow us to decouple our deploys from app releases, for e.g if we want to rollback manually. Any commit to the main branch on the configuration repo should trigger this workflow.

So, we need to run `kubectl` from Github Actions, but we obviously don't want to give it admin access -- it should only be able to do deployment related operations (principle of least privilege). As such, we can't use the same kube config as our local machine. Inspecting the `users` section from the config:
```yaml
users:
- name: do-blr1-k8s-learning-cluster-admin
  user:
    token: ****
```

The DO user is authenticating using a token and has admin privileges. We need to create a new user and restrict its permissions.

### Creating a CI user
Kubernetes supports 2 kinds of accounts - service accounts and user accounts. Service accounts are meant to provide identity for processes running *within* the cluster, so in general it's a bad idea to use them for accessing the cluster from outside. For our purposes, a *CI* user account should do.

Kubernetes also supports different authentication mechanisms - X509 client certs and static tokens. Static tokens need to be provided in a file to the K8s API server, and since we don't have direct access to the managed DO API server, we'll have to make do with X509 client certs. These certs need to be signed by the root CA that was created during K8s bootstrap.

To do this, we'll need to generate a Certificate Signing Request (CSR) and submit it to the K8s API. Let's generate a CSR using the excellent [cfssl](https://github.com/cloudflare/cfssl) tool which allows us to avoid the magic incantions required by `openssl`. Let's create a json spec for the CSR in `k8s-demo-ci-csr.json`:

```json
{
  "CN": "k8s-demo-ci",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "San Francisco",
      "O": "bots",
      "OU": "Juggernaut k8s demo",
      "ST": "CA"
    }
  ]
}
```

Generate the private key and CSR:
```bash
$ cfssl genkey k8s-demo-ci-csr.json | cfssljson -bare k8s-demo-ci
```

This generates a .csr and a .pem key file. Now, submit the CSR to the K8s API for approval:

```bash
$ cat << EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: k8s-demo-ci
spec:
  request: $(cat ./k8s-demo-ci.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages:
     - client auth
EOF
```

The CSR needs approval - you can do so yourself as the admin:
```bash
$ kubectl certificate approve k8s-demo-ci
```

This kicks off the signing process. Verify that the cert was signed and generated:
```bash
kubectl get csr k8s-demo-ci -o jsonpath='{.status.certificate}'| base64 -d > k8s-demo-ci.crt
```

### Limiting permissions
We want to limit the new `k8s-demo-ci` user's permissions to be able to do deployment-related operations only. Kubernetes provides 2 ways for access control: RBAC and ABAC. We'll use RBAC here. Let's create a role that only gives access to operations on the `deployments` API:

```bash
$ cat << EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: deployment-admin
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

Then, we'll assign (or bind) our `k8s-demo-ci` user to this role:
```bash
$ kubectl create rolebinding k8s-demo-ci-binding --role=deployment-admin --user=k8s-demo-ci --namespace=default
```

To verify, copy the original kube config as `k8s-demo-ci-kubeconfig` and replace the users section with the following:
```yaml
users:
- name: k8s-demo-ci
  user:
   client-certificate-data: <base64-encoded cert>
   client-key-data: <base64-encoded key>
```

The base64 encoded cert can be downloaded from the API directly:
```bash
$ kubectl get csr k8s-demo-ci -o jsonpath='{.status.certificate}'
```

And base64 encode the key data (which we generated using `cfssl`) yourself:
```bash
$ base64 k8s-demo-ci-key.pem
```

Now, pass this custom kube config to `kubectl`:
```bash
$ kubectl --kubeconfig k8s-demo-ci-kubeconfig get deployments
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
k8s-demo-go-app-deployment   1/1     1            1           5d23h
```

You can also verify that the new user doesn't have access to other resources:
```bash
$ kubectl --kubeconfig k8s-demo-ci-kubeconfig cluster-info

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
Error from server (Forbidden): services is forbidden: User "k8s-demo-ci" cannot list resource "services" in API group "" in the namespace "kube-system"
```

### Running kubectl from Github Actions
We have all the pieces ready, but we don't want to check in the kube config for the `k8s-demo-ci` user in Git since it contains the private key! We want to be able store the cert and key data as secrets and access them during the workflow run. We _could_ use a templating language or do some crude templating ourselves to insert the secrets at workflow run time. I ended up doing the simplest thing and going with base64-encoding the entire kubeconfig and storing it as a secret. Then, we can pass the kube config using bash process substitution:
```bash
kubectl --kubeconfig <(echo "$KUBECONF_SECRET_DATA" | base64 -d) apply -k .
```

Putting it all together, the workflow in the config repo:
```yaml
name: Deploy

on:
  push:
    branches: [ main ]
  
  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - name: Install kubectl
        run: |
         curl -LO https://dl.k8s.io/release/v1.21.3/bin/linux/amd64/kubectl
         chmod +x kubectl

      - name: Deploy to k8s cluster
        env:
          KUBECONF_DATA: ${{ secrets.KUBECONF_DATA }}
        run: ./kubectl --kubeconfig <(echo "$KUBECONF_DATA" | base64 -d) apply -k .
```

## Conclusion

There were several K8s concepts I needed to learn to set up even a simple deployment pipeline. 

Oh, by the way, and it's obvious, but it needs to be said that this is not something you should use for a production pipeline. I took several shortcuts and hacked my way through making it work. That said, it was a super useful exercise for me to get my feet wet with K8s. 

There are some flaws here - relying on plumbing between Github action workflows means that failure of any step of the workflow run would fail the deploy. Worse, it could leave our repos in an inconsistent state w.r.t to the actual cluster state. 

In future blog posts, I plan to explore Kubernetes-native solutions for creating CI/CD pipelines like [ArgoCD](https://argo-cd.readthedocs.io/en/stable/)


[^1]: Personally, I'm not a big fan of "deploy on every commit" and believe that in practice you need some manual control over the release process.