---
layout: post
title:  "Nginx+Flask+Postgres multi-container setup with Docker Compose"
date:   2017-09-18 18:40:23
categories: "docker"
---
In my [previous post]({{ site.baseurl }}{% post_url 2017-09-14-docker-migrating-legacy-links %}), I wrote about how I migrated my app to use user-defined networks.
As I mentioned in that post, I preferred to start with just the basic docker commands to avoid "magic" as much as possible. However, as expected, running
multiple docker commands by hand or in a shell script is far too brittle. Docker Compose is the recommended tool to manage multi-container deployments.
For illustrative purposes, I've extracted a subset of my code and posted a fully functional example on GitHub. If you're the impatient type,
go here to try it out. 

## Recap

My 3-container setup is pictured below as explained in the [previous post]({{ site.baseurl }}{% post_url 2017-09-14-docker-migrating-legacy-links %}).

![Proposed Setup]({{ site.baseurl }}/assets/MigratingLegacyLinksProposedSetup.png)

## First attempt

`docker-compose` allows you to define all your containers, networks and volumes in a nice declarative yaml file. My first attempt
at `docker-compose.yml` looked something like this:

<pre>
version: '3'
services:
  db:
    image: "postgres:9.6.5"
    networks:
      - db_nw
  flaskapp:
    build: .
    volumes:
      - .:/opt/services/flaskapp/src
    networks:
      - db_nw
      - web_nw
  nginx:
    image: "nginx:1.13.5"
    ports:
      - "8080:80"
    volumes:
      - ./conf.d:/etc/nginx/conf.d
    networks:
      - web_nw
networks:
  db_nw:
    driver: bridge
  web_nw:
    driver: bridge
</pre>

This looks like a fairly self-explanatory translation of the above diagram, but it doesn't work out-of-the-box yet.

## Bootstrapping the database

The first time we bring up the cluster with a `docker-compose up`, the database will not have the required tables created yet.
Fortunately, the postgres image allows you to specify a default user, password and database through environment variables.
We'll create an environment file with the required variables and specify it in the `db` section:

<pre>
services:
  db:
    image: "postgres:9.6.5"<b>
    env_file:
      - env_file</b>
    networks:
      - db_nw
</pre>

This still doesn't solve the issue of creating our database schema. Our flask app declares the `SQLAlchemy` models, so 
let's create the schema from them. Create a `database.py` with an `init_db` function to create the schema:

{% highlight python %}
# These env variables are the same ones used for the DB container
user = os.environ['POSTGRES_USER']
pwd = os.environ['POSTGRES_PASSWORD']
db = os.environ['POSTGRES_DB']
host = 'db' # docker-compose creates a hostname alias with the service name
port = '5432' # default postgres port 
engine = create_engine('postgres://%s:%s@%s:%s/%s' % (user, pwd, host, port, db)) 

Base = declarative_base()

def init_db():
    # import all modules here that might define models so that
    # they will be registered properly on the metadata.  Otherwise
    # you will have to import them first before calling init_db()
    import models
    Base.metadata.create_all(bind=engine)
{% endhighlight %}

Remember though, that our flask app lives in a separate container so we'll have to
figure out a way to connect it to the DB container and run a one-off command to create the schema.

Let's first bring up only the DB container:
{% highlight bash %}
$ docker-compose up -d db
{% endhighlight %}
Notice the refreshing lack of `--env-file` or `--network` flags we would have had to pass in a plain `docker run` command.
This is because we've already specified those in the `docker-compose.yml` file.

Now, we'll run a one-off flask container that will create the schema:
{% highlight bash %}
$ docker-compose run --rm flaskapp /bin/bash -c "cd /opt/services/flaskapp/src && python -c  'import database; database.init_db()'"
{% endhighlight %}

Notice the `--rm` flag that indicates the container should be deleted immediately after completion. 

### A note about volumes

The [documentation](https://docs.docker.com/compose/overview/#preserve-volume-data-when-containers-are-created) 
 for `docker-compose` states that volume data is preserved across runs. Quoting from the docs:
> When `docker-compose` up runs, if it finds any containers from previous runs, it copies the volumes from the old container to the new container. This process ensures that any data you’ve created in volumes isn’t lost.

 So imagine my surprise when I did a `docker-compose down && docker-compose up -d` and found out that the database schema I created earlier
was lost! Turns out that volume data is copied across runs only for _named volumes_; since we didn't specify a named volume mount in the compose yaml,
`docker-compose` creates a new anonymous volume each time the `db` container is started. To remedy this, we'll mount a named volume.
Additions are highlighted in bold:

<pre>
services:
  db:
    image: "postgres:9.6.5"<b>
    volumes:
      - "dbdata:/var/lib/postgresql/data"</b>
    env_file:
      - env_file
    networks:
      - db_nw<b>
volumes:
  dbdata:</b>
</pre>

Note that you need to declare the volume at the top-level to reference it in the `db` service section.
Now we see that the database volume data is preserved across runs. If you want to actually delete the volume,
pass the `--volumes` option to `docker-compose down`

## Container startup order

Our `docker-compose.yml` is still not quite ready yet - `docker-compose up` attempts to start containers in parallel,
meaning that our flask container could come up before the db container, and die because it can't find the db. We'll
tell docker-compose to start the `db` container before the `flaskapp` container by specifying a dependency:

<pre>
  flaskapp:
    build: .
    env_file:
      - env_file
    volumes:
      - .:/opt/services/flaskapp/src
    networks:
      - db_nw
      - web_nw<b>
    depends_on:
      - db</b>
</pre>

In theory, this doesn't actually solve the "race condition" because the postgres process could take a while to start up
and docker has no way of knowing when it's "done" starting up. However, in my experience, it works well enough in practice
for this example.

## Putting it all together

The final piece is to hook up our nginx reverse proxy to the flask container. We'll use this very simple config to proxy
all requests to flask:

<pre>
server {
    listen 80;
    server_name localhost;

    location / {
        proxy_set_header   Host                 $host;
        proxy_set_header   X-Real-IP            $remote_addr;
        proxy_set_header   X-Forwarded-For      $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto    $scheme;
        proxy_set_header Host $http_host;

        proxy_pass http://flaskapp:5090;
    }
}
</pre>

Notice that the hostname in `proxy_pass` is just the service name, `flaskapp`. Docker's embedded DNS server
will automagically resolve it to the IP of the actual container(s) running the service.

The nginx container will be the only "public" entry point to our cluster, so let's map it to port 8080 of the host.

<pre>
  nginx:
    image: "nginx:1.13.5"<b>
    ports:
      - "8080:80"</b>
    volumes:
      - ./conf.d:/etc/nginx/conf.d
    networks:
      - web_nw
</pre>

Finally, we'll bring up the cluster with
{% highlight bash %}
$ docker-compose up -d
{% endhighlight %}

Browse to localhost:8080 in your browser, and you should see the "Hello, please sign up!" message.

## Summary

`docker-compose` provides a robust, repeatable way to manage multi-container clusters. It also provides the
ability to easily scale your services. For example, to scale the flaskapp service to 2 containers, just run:

{% highlight bash %}
$ docker-compose up -d --scale flaskapp=2
{% endhighlight %}

This will start a second container running flaskapp. In theory, nginx _should_ automatically round-robin requests
between the 2 containers. But does it actually? Find out in my next post 😁
