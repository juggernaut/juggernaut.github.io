---
layout: post
title:  "Dynamic Nginx configuration for Docker with Python"
date:   2017-09-27 18:40:23
categories: "docker"
---
In my [previous post]({{ site.baseurl }}{% post_url 2017-09-20-nginx-flask-postgres-docker-compose %}), I wrote about my multi-container setup with `docker-compose`.
One of the benefits of `docker-compose` is that it makes it easy to scale; for example, to scale the `web` service, you can simply run:
{% highlight bash %}
$ docker-compose up -d --scale web=2
{% endhighlight %}
If the `web` service is currently running 1 container, `docker-compose` will spin up an additional one. If you have nginx fronting the web
service (like in my setup), we'd expect nginx to round-robin requests between the two `web` containers, right?

Let's check it for ourselves. Recall how the nginx config looks like:

<pre>
server {
    listen 80;
    server_name localhost;

    location / {
        ....

        proxy_pass http://flaskapp:5090;
    }
}
</pre>

where `flaskapp` is the service name in `docker-compose.yml`. Docker's embedded DNS server resolves the service name to the actual container IPs.
It implements DNS round-robin, so a client sees the list of IPs shuffled each time it resolves the service name. Let's confirm this.
Assuming `11d3838afca6c` is the nginx container id:
{% highlight bash %}
$ docker exec -it 11d3838afca6 /bin/bash

root@11d3838afca6:/# dig +short flaskapp
172.21.0.2
172.21.0.4
root@11d3838afca6:/# dig +short flaskapp
172.21.0.4
172.21.0.2
{% endhighlight %}

Cool, that works. Now, let's run some curl requests to make sure that the HTTP requests made via nginx are indeed round-robined across the 2 containers:
{% highlight bash %}
$ for i in `seq 5`
> do
> curl -s -o /dev/null localhost:8080
> done

$ docker-compose logs -f flaskapp

flaskapp_1  | 172.21.0.3 - - [26/Sep/2017 06:56:03] "GET / HTTP/1.0" 200 -
flaskapp_1  | 172.21.0.3 - - [26/Sep/2017 06:56:03] "GET / HTTP/1.0" 200 -
flaskapp_1  | 172.21.0.3 - - [26/Sep/2017 06:56:03] "GET / HTTP/1.0" 200 -
flaskapp_1  | 172.21.0.3 - - [26/Sep/2017 06:56:03] "GET / HTTP/1.0" 200 -
flaskapp_1  | 172.21.0.3 - - [26/Sep/2017 06:56:03] "GET / HTTP/1.0" 200 -
{% endhighlight %}

Wait, what!? All requests seem to be going to `flaskapp_1` ! Turns out nginx caches DNS resolutions until the next reload. [This excellent post](
https://tenzer.dk/nginx-with-dynamic-upstreams/) goes into detail about how this works and suggests some workarounds.

## The Workaround

_TL;DR_ of the blog post mentioned - if you want to avoid a hefty $2000 per instance license for _NGINX Plus_, write your configuration like this:

<pre>
<b>resolver 127.0.0.11;
set $backends flaskapp</b>
location / {
    ....
    proxy_pass http://$backends:5090;
}
</pre>

Note that `127.0.0.11` is the IP of Docker's embedded DNS server. Using variables forces resolution through a `resolver` which we point at the embedded
DNS server.

The resulting request distribution looks much more equitable now:
<pre>
flaskapp_1  | 172.21.0.3 - - [27/Sep/2017 04:52:08] "GET / HTTP/1.0" 200 -
flaskapp_2  | 172.21.0.3 - - [27/Sep/2017 04:52:08] "GET / HTTP/1.0" 200 -
flaskapp_1  | 172.21.0.3 - - [27/Sep/2017 04:52:08] "GET / HTTP/1.0" 200 -
flaskapp_2  | 172.21.0.3 - - [27/Sep/2017 04:52:08] "GET / HTTP/1.0" 200 -
flaskapp_1  | 172.21.0.3 - - [27/Sep/2017 04:52:08] "GET / HTTP/1.0" 200 -
</pre>

## Can we do better?

Like all workarounds, there are drawbacks to the above approach. Sine we can't specify an `upstream` block, we lose all the nice features
provided by nginx's [upstream module](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#upstream) like load-balancing policies,
weights and health checks.

If we could somehow update the `upstream` list dynamically as docker adds/removes containers, then reload nginx,
we could have the best of both worlds. Docker provides a handy [event stream](https://docs.docker.com/engine/reference/commandline/events/) 
that we can hook into and listen for container lifecycle events. There are tools out there that already do this, which leads me to my
disclaimer:

_**DISCLAIMER**_: <em>The following section talks about a python script I wrote merely for learning purposes. It is not production-quality code and serious deployments should use widely-used tools like [traefik](https://traefik.io/) and [nginx-proxy](https://github.com/jwilder/nginx-proxy)</em>

With that out of the way, let's start by installing the [docker python sdk](https://docker-py.readthedocs.io/en/stable/):
{% highlight bash %}
pip install docker
{% endhighlight %}

We want to listen for container lifecycle events, so let's start with a skeleton to capture those:
{% highlight python %}
event_filters = {'type': 'container'}
for event in client.events(filters=event_filters, decode=True):
    if event['status'] == 'start':
        # update container list and reload nginx
    elif event['status'] == 'stop':
        # update container list and reload nginx
{% endhighlight %}

But this captures events for _all_ containers managed by our system Docker. We only want to capture container events for our `web` service.
One way could be to filter by container name - as of version 1.14.0 of _docker-compose_ container names seem to follow a format of
_$project\_$service\_$index_, so if we have 2 containers of the `flaskapp` service in a project called `myproj`, they would have
names `myproj_flaskapp_1` and `myproj_flaskapp_2`. However, relying on implementation details seems wrong; there must be a better way.

### Introducing labels

Labels are key-value metadata that can be attached to Docker objects including containers, images and even networks. If we annotate
services with custom labels, we can reliably identify them from the event stream. Let's add a label to our `nginx` and `flaskapp`
services in _docker-compose.yml_ (for the full _docker-compose.yml_, refer to my [previous post]({{ site.baseurl }}{% post_url 2017-09-20-nginx-flask-postgres-docker-compose %})):
<pre>
flaskapp:
  ...
  <b>labels:
    com.ameyalokare.type: web</b>
nginx:
  ...
  <b>labels:
    com.ameyalokare.type: nginx</b>
</pre>

Now we can update the `event_filters` to include a label:
<pre>
event_filters = {'type': 'container', <b>'label': 'com.ameyalokare.type=web'</b>}
</pre>

### Updating upstream server list

Our python script needs to maintain a list of currently running `web` containers, and if this list changes, we'll reload nginx.
Let's use a _dict_ mapping container_ids to container_objects for this:

{% highlight python %}
web_servers = {}
event_filters = {'type': 'container', 'label': 'com.ameyalokare.type=web'}
for event in client.events(filters=event_filters, decode=True):
    if event['status'] == 'start':
        web_servers[event['id']] = client.containers.get(event['id'])
    elif event['status'] == 'stop':
        del web_servers[event['id']]
    else:
        continue
    print "Received {} event for container {}".format(event['status'], event['id'])
    # TO BE IMPLEMENTED
    # update_config_and_reload_nginx(client, web_servers)
{% endhighlight %}
