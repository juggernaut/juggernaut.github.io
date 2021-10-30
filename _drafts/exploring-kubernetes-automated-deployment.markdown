---
layout: post
title:  "Exploring Kubernetes: Building an automated deployment pipeline"
date:   2021-10-30 18:40:23
categories: technology kubernetes ci/cd github-actions
---

I'd been hearing about this hot technology called "Kubernetes" for years, but never really looked into it in earnest. Now that the madness seems to have died down and Kubernetes is "so 2017", I decided it was time to dive into it.

My previous employer, Twilio, was a pretty early adopter of the cloud. They went all in on AWS in 2008, back when "cloud native" was not a thing. When I joined in 2012, we still hadn't completely figured out the best practices around building software in the cloud -- over the next few years, the engineering org spent a lot of time improving the platform infrastructure around building, deploying and operating our software. As a product engineer with a keen interest in infrastructure, I was a witness to (and occasional contributor) as we built out this tooling.

 Around the same time, several companies were having similar problems and trying to solve them in similar ways. A consensus was emerging around best practices -- immutable infrastructure, blue-green deployments, stateless-as-possible services etc. A lot of these best practices were built into Twilio's custom platform, and I believe, several other companies as well. This meant that a lot of similar work was being done at different places. In the software world, such a situation has almost always resulted in the standardization of a single player who does the "undifferentiated heavy lifting". As I understand it, Kubernetes is this standard. There are benefits to standardization - developers can onboard quicker since they don't have to learn one more deployment and orchestration system. It also gives ops folks skill portability and reduces their support burden, since silly questions from devs can now be "outsourced" to the open source community.

 That said, I wanted to learn for myself how exactly Kubernetes works and whether it delivers on its promises. One problem with the k8s community is that it's full of buzzwords. There are a zillion third party tools to do everything, and I wanted to take more of a "bottom-up" approach using vanilla Kubernetes as much as possible. I learn best by doing, so I fired up a k8s cluster, and went through a few toy exercises which I document in this series of blog posts.

 ## Exercise: Build an automated deployment pipeline

I figured a simple but instructive exercise would be to set up an automated deployment for a Go application. Github Actions is super easy (and free) to get started, so I'm going to use it as the CI/CD platform. I actually went through the exercise of setting up a Kubernetes cluster from scratch using TODO:LKHTHW, but I'm not really focused on the admin aspect here, so a managed k8s service is good enough. I toyed with Google Kubernetes Engine (GKE) but couldn't wrap my head around their pricing, so I settled for Digital Ocean which costs $10/month for a minimal cluster.

### Booting the k8s cluster
DO has an intuitive interface to start a k8s cluster. I chose a 1 node control plane, 1 node worker cluster -- we're not really worried about HA here. It took a few minutes to boot. DO provides a "kubeconfig" file which contains the settings to access the cluster. We'll need to copy this file to `~/.kube/config`

Let's make sure we can connect to the cluster:

```
$ kubectl cluster-info
Kubernetes master is running at https://d24b5f0b-6900-4caf-9b5c-7739e2c70bbc.k8s.ondigitalocean.com
CoreDNS is running at https://d24b5f0b-6900-4caf-9b5c-7739e2c70bbc.k8s.ondigitalocean.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

All good.

### Sample app
I created a simple Go app with a `/demo` endpoint that returns a welcome message. I dockerized it and pushed it to Docker Hub. TODO: find it here

### Manual deployment

Let's try deploying the app manually first. APIs in Kubernetes are declarative - you specify a *deployment resource* and the engine goes and figures out how to make it happen.

Create a `deployment.yaml`:

```yaml
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

The yaml is fairly self-explanatory. The important parts are `replicas` which specifies the number of instances of your app you want to run, and `image` which should be the fully qualified docker image name to be pulled.

Let's deploy it:
```
kubectl apply -f deployment.yaml
```

It takes a second to roll out:
```
kubectl rollout status deployment/k8s-demo-go-app-deployment
TODO: output here
```

Kubernetes has now deployed a single *pod* on our worker node. A *pod* is k8s' unit of deployment -- it *can* house multiple containers, but usually contains just one. 

### Verifying the deployment

The deployed app isn't really accessible publicly - we'll skip doing this for now and access it locally.

Let's get the pod name
```
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
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

Let's go through another iteration of a manual deployment to update the app. Let's first change the message returned by the app, and tag the docker image with the next version (v0.0.2). 

TODO: docker commands to do this.

TODO: kubectl apply again after changing image tag

A new pod has been deployed and the old pod has been removed. This is pretty nice, immutable deployments FTW!

### Automating the process

Now, I want to set up a Github Actions pipeline to support the following workflow:
1. I create a release tag on Github
2. Automatically kick off a docker build, tag and push to Docker hub
3. Bump the image tag in the kubernetes deployment resource
4. Apply the deployment

1\. and 2. is standard docker stuff, so I won't cover it. For 3. and 4. I created a separate git repo to house the kubernetes yaml resources. I *could* have put it in the app repo, but tying the app version tightly to the deployment means that you can't easily manually rollback. The idea is to have a separate config repo that defines the state of the cluster. Any update to that repo should trigger a `kubectl apply` to the cluster. This is what the cool kids are calling "GitOps" these days.

### Bumping Docker image version

Inspecting the yaml, I briefly considered unix tools like sed/awk to update the image version, but decided it was too unwieldy. Unsurprisingly, the k8s community has come up with their own solution to update these yaml files - `kustomize`. It's a declarative way to specify patches on top of the yaml. `kubectl` has native support for applying `kustomize` patches, so you don't need to explicitly modify the original yaml.

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

### Tying it all together

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

This workflow runs when I create a release on Github and currently just checks out the repo. To do "real" continuous delivery, you'll want to hook it up to run on *any* push to master [1]. 

Next, let's build and push the Docker app image. We'll need an access token for Docker Hub which you can create by following the instructions [here](https://docs.docker.com/docker-hub/access-tokens/). Make sure to add the Docker Hub username and access token as [secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets) in Github actions to avoid leaking them.

```yaml
- name: Build the k8s demo app Docker image
  run: docker build . --file Dockerfile --tag juggernaut/k8s-demo-go-app:${{ github.event.release.tag_name }}

- name: Push the k8s demo app Docker image to Docker Hub
  run: docker push juggernaut/k8s-demo-go-app:${{ github.event.release.tag_name }}
```


^[1]: Personally, I'm not a big fan of "deploy on every commit" and believe that in practice you need some manual control over the release process.