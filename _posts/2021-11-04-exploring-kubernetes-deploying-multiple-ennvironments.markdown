---
layout: post
title:  "Exploring Kubernetes: Deploying in multiple environments"
date:   2021-11-03 14:10:32
categories: technology kubernetes
---

In [part 1](https://www.ameyalokare.com/technology/kubernetes/ci/cd/github-actions/2021/10/30/exploring-kubernetes-automated-deployment-part-1.html), I set up an automated deployment process for a simple app on Kubernetes. There was a single environment in which to deploy the app. In the real world, you'd at least need an environment to test changes (`staging`) in addition to the actual `production` environment. Likely, you'll also need a `development` environment for developers to be able to run experimental changes.

In this post, I explore deploying a sample app in multiple environments. In the process, I also learned about how Kubernetes supports managing configuration data (`ConfigMap`s) and secrets.

## Sample app

The app in [part 1](https://www.ameyalokare.com/technology/kubernetes/ci/cd/github-actions/2021/10/30/exploring-kubernetes-automated-deployment-part-1.html) was *too* simple. To make it a little more representative of the real world, I modified the app to serve two API endpoints - `POST /employees` and `GET /employees`. It stores `employee`s in a database. The database driver and connection string are taken from environment variables. This will allow us to test with sqlite locally (e.g see [unit test](https://github.com/juggernaut/k8s-demo-emp-api/blob/d1fa876c2fa9dfc9b4d54e874597291aac883a29/api_test.go#L39)) and use postgres in the cluster. Find the [sample app on github](https://github.com/juggernaut/k8s-demo-emp-api).

## Namespaces

Kubernetes supports *namespaces* which provide a named scope for kubernetes resources. Most resources can be created within a namespace [^1]. There is a `default` namespace which is used by default (duh) when you don't specify a namespace. This looks like it fits our use case of multiple isolated environments. Let's go ahead and create two namespaces `staging` and `production`.

> **_WARNING:_** Even though namespaces may functionally isolate resources, the underlying compute, I/O and storage are shared -- they're part of the same cluster after all. As such, I wouldn't recommend running the production environment in a namespace. Make a separate cluster for it. Take this post as a learning exercise, not what you'd want to actually do in production.

{% highlight bash %}
$ kubectl create namespace staging
$ kubectl create namespace production
{% endhighlight %}

## Setting up the DB

Let's spin up a DigitalOcean managed Postgres database. The cheapest option at $15/month is sufficient for our toy use case. Make sure we only allow incoming connections from our k8s cluster [^2]. This is pretty intuitive to do from the UI. DO creates a default user/database and enables SSL by default which is nice. After it's done spinning up the DB, you should get a set of parameters to connect to it including the password and the root CA certificate used to sign the DB certificate -- we'll need these, so download them.

To isolate environments, we'll create separate logical DBs within postgres [^3]. Again, in a real production use case, you'll want to spin up a separate DB instance for each environment. First, let's create a user and logical DB for the `staging` environment. Remember, we configured our instance to only allow connections from the k8s cluster, so to do this we'll need to run a pod in the cluster:

{% highlight bash %}
$ kubectl run -i --rm --env="PGPASSWORD=<redacted>" --tty postgres \
  --image=postgres --restart=Never -- \
  psql \
    -h private-db-postgresql-blr1-46219-do-user-998612-0.b.db.ondigitalocean.com \
    -U doadmin -p 25060 defaultdb
{% endhighlight %}

Notice that the command is similar to `docker run` which is nice to run a one-off pod in the cluster for admin tasks like this.

This drops you into a postgres shell. Create a postgres role for the API user, the database and `employees` table, then grant the API user access:

{% highlight sql %}
defaultdb=> CREATE USER apiuser_staging WITH ENCRYPTED PASSWORD '<redacted>';
defaultdb=> CREATE DATABASE acmedb_staging;
defaultdb=> \c acmedb_staging;
acmedb_staging=> CREATE TABLE employees (name varchar(256) NOT NULL, age int NOT NULL);
acmedb_staging=> GRANT SELECT, UPDATE, INSERT, DELETE ON employees TO apiuser_staging;
{% endhighlight %}

Exit the shell, and you'll see a message that the pod was deleted (the `--rm` flag ensures this).

Great, the DB is now ready for use in the `staging` environment.

## Securing the database connection

You'll notice when we connected to the DB in the above commands, we didn't specify anything related to SSL or verifying certificates. The default [sslmode](https://www.postgresql.org/docs/8.4/libpq-connect.html#LIBPQ-CONNECT-SSLMODE) is `prefer` which means it'll try a SSL connection and if that fails, it'll try a non-SSL connection. This is dangerously insecure because it leaves us vulnerable to downgrade attacks. The `require` mode is not sufficient either, since it does not verify the CA certificate if left unspecified; this opens us up to man-in-the-middle attacks. The only really secure mode is `verify-full` which verifies the CA certificate as well as the hostname. Luckily, DO has provided us with the CA cert, so we need to pass `"sslmode=verify-full sslrootcert=<path to CA cert>"`.

### ConfigMap for storing certificate
So, now our problem is, how do we get the certificate accessible to the application pod? Enter *configmap*s. A *configmap* is a bag of configuration data that can be consumed by pods in various ways. They can be scoped to a namespace, so we can specify different values for `staging` and `production`. A thing to keep in mind is that *configmap*s are **not** for secret data -- so don't store your API tokens and secret keys in configmaps.

> **_NOTE_**: The root CA certificate is not secret data. It does not contain the private key.

Let's create a configmap scoped to `staging` from the CA certificate:
{% highlight bash %}
$ kubectl -n=staging create configmap dbparams --from-file=cacertificate.crt
{% endhighlight %}

Now, see that the configmap was created with a single key `cacertificate.crt` with the value being the certificate data:

{% highlight bash %}
$ kubectl -n=staging describe configmap dbparams
Name:         dbparams
Namespace:    staging
Labels:       <none>
Annotations:  <none>

Data
====
cacertificate.crt:
----
-----BEGIN CERTIFICATE-----
MIIEQTCCAqmgAwIBAgIUeMBS3Ap7SD2PwaeiGlRQqs5TD1UwDQYJKoZIhvcNAQEM
BQAwOjE4MDYGA1UEAwwvMzRlMzM2ZjQtNzRlYi00Y2JjLWEyNWYtZjgxNGExODU4
......
-----END CERTIFICATE-----
{% endhighlight %}

### Mounting the ConfigMap

Now, let's specify a deployment for our app, specifying the `DB_DRIVER_NAME` and `DB_CONNECTION_STRING` environment variables: 

{% highlight yaml %}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-demo-emp-api-deployment
spec:
  selector:
    matchLabels:
      app: k8s-demo-emp-api
  replicas: 1
  template:
    metadata:
      labels:
        app: k8s-demo-emp-api
    spec:
      containers:
      - name: k8s-demo-emp-api
        image: juggernaut/k8s-demo-emp-api:latest
        env:
          - name: "DB_DRIVER_NAME"
            value: "postgres"
          - name: "DB_CONNECTION_STRING"
            value: "host=private-db-postgresql-blr1-46219-do-user-998612-0.b.db.ondigitalocean.com port=25060 user=api_staging password=<redacted> dbname=acmedb_staging sslmode=verify-full sslrootcert=/dbparams/cacertificate.crt"
        volumeMounts:
          - name: "dbparams"
            mountPath: "/dbparams"
            readOnly: true
        ports:
        - containerPort: 9090
      volumes:
        - name: "dbparams"
          configMap:
            name: "dbparams"
{% endhighlight %}

The interesting bits here are `volumes` and `volumeMounts`. You can specify a volume with a configmap as its source -- each key in the configmap becomes a file. In `volumeMounts`, you specify where in the filesystem you want to mount it (`/dbparams`). With the mount in place, we can pass `sslrootcert=/dbparams/cacertificate.crt` in the connection string.

`kubectl apply` the deployment and our app now has a secure connection to the DB.

## Securing the password

Notice that we're passing around the DB password in plain-text -- this is bad practice. The yaml resources will be checked into source control as part of our GitOps workflow, and checking in secrets is a bad, bad idea.

Kubernetes can store secrets for us and pass them safely to pods. A vanilla K8s install stores secrets unencrypted (which is batshit crazy IMO). Most managed providers including DO K8s encrypts secrets by default.

### Secrets

Let's store our DB credentials in a secret:
{% highlight bash %}
$ kubectl -n=staging create secret generic db-secret \
  --from-literal="db_user=apiuser_staging" --from-literal='db_password=<redacted>'
{% endhighlight %}

Similar to configmaps, there are various ways to consume secrets in pods. We'll use environment variables. Let's update the `env` section:

{% highlight yaml %}
env:
  - name: "DB_USER"
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: db_user
  - name: "DB_PASSWORD"
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: db_password
{% endhighlight %}


We've defined two env variables `DB_USER` and `DB_PASSWORD` which source their values from the secret `db-secret`. Simple enough.

### Dependent environment variables

Notice that our app doesn't actually take `DB_USER` and `DB_PASSWORD` environment variables; it takes a DB connection string. We'll need to somehow refer to the user and password variables in `DB_CONNECTION_STRING`. Fortunately, Kubernetes has just the thing - [dependent environment variables](https://kubernetes.io/docs/tasks/inject-data-application/define-interdependent-environment-variables/). If you use $(VAR), K8s will substitute the value of $VAR. Let's define the connection string then:

{% highlight yaml %}
- name: "DB_CONNECTION_STRING"
  value: >-
    host=private-db-postgresql-blr1-46219-do-user-998612-0.b.db.ondigitalocean.com
     port=25060 user=$(DB_USER) password=$(DB_PASSWORD) dbname=acmedb_staging
      sslmode=verify-full sslrootcert=/dbparams/cacertificate.crt
{% endhighlight %}

We're now securely passing the password to our application.

## Environment variables from ConfigMap

There are a couple more kinks to iron out. The `dbname` is hard-coded, so it wouldn't work in the `production` environment. We'll need to extract it into an environment variable. In the real world, the db host/port would also be different for staging vs production, so that needs to be parameterized as well. Earlier, we saw how to mount a `ConfigMap` as a volume; now let's see how to consume it as environment variables. First, let's edit our `dbparams` configmap:

{% highlight bash %}
$ kubectl -n=staging edit configmap dbparams
{% endhighlight %}

This opens the configmap in your $EDITOR. Go ahead and add a key `db_name` with value "acmedb_staging" and save it. We can now create an environment variable sourced from this key in `deployment.yaml`:

{% highlight yaml %}
- name: "DB_NAME"
  valueFrom:
    configMapKeyRef:
      name: dbparams
      key: db_name
{% endhighlight %}

Let's do the same for DB_HOST and DB_PORT. Now we have a nicely parameterized deployment resource that can be applied to any environment. The full deployment.yaml looks like:

{% highlight bash %}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-demo-emp-api-deployment
spec:
  selector:
    matchLabels:
      app: k8s-demo-emp-api
  replicas: 1
  template:
    metadata:
      labels:
        app: k8s-demo-emp-api
    spec:
      containers:
      - name: k8s-demo-emp-api
        image: juggernaut/k8s-demo-emp-api:latest
        env:
          - name: "DB_USER"
            valueFrom:
              secretKeyRef:
                name: db-secret
                key: db_user
          - name: "DB_PASSWORD"
            valueFrom:
              secretKeyRef:
                name: db-secret
                key: db_password
          - name: "DB_NAME"
            valueFrom:
              configMapKeyRef:
                name: dbparams
                key: db_name
          - name: "DB_HOST"
            valueFrom:
              configMapKeyRef:
                name: dbparams
                key: db_host
          - name: "DB_PORT"
            valueFrom:
              configMapKeyRef:
                name: dbparams
                key: db_port
          - name: "DB_DRIVER_NAME"
            value: "postgres"
          - name: "DB_CONNECTION_STRING"
            value: "host=$(DB_HOST) port=$(DB_PORT) user=$(DB_USER) password=$(DB_PASSWORD) dbname=$(DB_NAME) sslmode=verify-full sslrootcert=/dbparams/cacertificate.crt"
        volumeMounts:
          - name: "dbparams"
            mountPath: "/dbparams"
            readOnly: true
        ports:
        - containerPort: 9090
      volumes:
        - name: "dbparams"
          configMap:
            name: "dbparams"
{% endhighlight %}

## Deploying to production

We've already done all the heavy lifting to make our app ready to deploy in any environment. We'll need to create the `acmedb_prod` logical DB and with the `apiuser_prod` postgres user similar to `staging`. We'll also need to create the `dbparams` configmap and the username/password secret in the `production` namespace (remember configmaps are scoped to namespaces):

{% highlight bash %}
# Create the dbparams configmap in one go, with values sourced from a file as well as literal values
$ kubectl -n=production create configmap dbparams \
   --from-file=cacertificate.crt \
   --from-literal=db_host=private-db-postgresql-blr1-46219-do-user-998612-0.b.db.ondigitalocean.com \
   --from-literal=db_port=25060
$ kubectl -n=staging create secret generic db-secret \
  --from-literal="db_user=apiuser_prod" --from-literal='db_password=<redacted>'
{% endhighlight %}

Now, we can apply the same deployment resource without changes!
{% highlight bash %}
$ kubectl -n=production apply -f deployment.yaml
{% endhighlight %}

## Conclusion

We learned about namespaces to create scoped resources. We learned how to utilize configmaps to cleanly separate code from configuration, allowing us to easily deploy in multiple environments. Additionally, we utilized secrets to securely pass confidential data to our app.

That said, I wouldn't use namespaces as an isolation mechanism in real production as the underlying compute is shared; I'd create a separate cluster instead. Maybe dev/staging can use a shared cluster with namespaces.

Re: secrets, I'd be extremely careful to make sure that they're configured to be encrypted with KMS (which is the default in Google Kubernetes Engine). To be honest, I wouldn't recommend using secrets in the DO managed K8s, as there is [little clarity](https://www.digitalocean.com/community/questions/are-kubernetes-secrets-encrypted-on-disk) on which encryption provider is used.

Additionally, Kubernetes secrets are static. Rotating them requires re-deploying your app to pick up the new values. There are 3rd party projects to automate this process, but that feels quite hacky to me. For short-lived/dynamic secret support, you'll want to skip K8s secrets altogether and utilize something like [Vault](https://www.vaultproject.io/).

In future blog posts, I'll explore extending my automated deployment process in [part 1](https://www.ameyalokare.com/technology/kubernetes/ci/cd/github-actions/2021/10/30/exploring-kubernetes-automated-deployment-part-1.html) to a proper CI/CD pipeline across staging and production environments.


[^1]: Some resources are cluster-wide or [may not be in any namespace at all](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/#not-all-objects-are-in-a-namespace)
[^2]: DO defaults to allow incoming connections from your local computer's IP. This is dangerous because that IP is quite likely shared between customers of your ISP.
[^3]: Using [Postgres Schemas](https://www.postgresql.org/docs/9.1/ddl-schemas.html) might be a cleaner solution here