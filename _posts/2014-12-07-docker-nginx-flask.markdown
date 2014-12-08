---
layout: post
title:  "Dockerizing nginx+flask app on OS X"
date:   2014-12-07 18:40:23
categories: docker nginx python
---
Recently, I bought a new Macbook Pro for my development needs. Migrating my development environment from my old Fedora-based box turned out to be a nightmare, mostly because I'd installed stuff willy-nilly with no way to repeatably recreate it. I decided to do it the Right Way<sup>TM</sup> this time. I've worked with [Vagrant](https://www.vagrantup.com/) before, but I decided to check out this Docker thing that the [Hacker News](https://news.ycombinator.com/) crowd is going ga-ga over.

##Requirements

My app is fairly standard - a python flask app fronted by an nginx reverse-proxy that also serves static content.

I had the following (reasonable) requirements:

+ **Quick local development cycle:** the code lives on my Macbook, and any changes to my code should auto restart the containerized app. *No* manual restarts required.
+ **Repeatable build process:** I want to be able to recreate my dev environment on other machines and preferably use the same build process to deploy on cloud platforms like AWS, GCE or DigitalOcean.

With all the talk about Docker, I couldn't find a simple step-by-step tutorial that shows you how to create such a setup, so here it is.

##Overview

Docker best practices recommend one process per container, so we will need 2 containers, one each for nginx and your flask app. In practice, you will need a third database container, but for simplification, I have omitted it.  Only the nginx container will be accessible to the outside world. The flask application container will only be accessible
from the nginx container; in fact, it doesn't even need network access. This process isolation is a major benefit touted by Docker.

## Step-by-step walkthrough

The objective of this tutorial is to quickly get you up and running, so I don't go into detail about what the various docker commands do.  I will assume that you have already set up Docker on OS X as detailed [here](https://docs.docker.com/installation/mac/). I will also assume you have built a flask application before and have the code handy on your Mac. Let's get started.

### **1. Create a Dockerfile for your Flask app**

The nice thing about Docker images is that they're extensible - you simply inherit most of the work that someone else has
already done for you and add your stuff to it. There is an official debian-based [Python image](https://registry.hub.docker.com/_/python/), however, I prefer running CentOS given that I'm familiar with the ecosystem, especially yum. I couldn't find a CentOS 6 image with Python 2.7.8, so I created one [here](https://registry.hub.docker.com/u/juggernaut/centos-python/).

Now, let's create a Dockerfile for your flask app. Create a file named `Dockerfile` in your flask app's root directory. I assume your app is called `flaskapp` and `app.py` runs it.

{% highlight ruby %}
FROM juggernaut/centos-python:centos6

RUN mkdir -p /opt/services/flaskapp/src
COPY . /opt/services/flaskapp/src
VOLUME ["/opt/services/flaskapp/src"]
WORKDIR /opt/services/flaskapp
RUN virtualenv-2.7 venv && source venv/bin/activate && pip install -r src/requirements.txt
EXPOSE 5000
CMD ["/opt/services/flaskapp/venv/bin/python2.7", "/opt/services/flaskapp/app.py"]
{% endhighlight %}

All this does is copy your source directory into the container, install virtualenv and exposes the default flask port (more on this later).

### **2. Build Flask app image**

{% highlight bash %}
docker build -t flaskapp .
{% endhighlight %}

This creates a docker image named `flaskapp`.

### **3. Run Flask app container**

{% highlight bash %}
docker run -d -P --name flaskapp -v $PWD:/opt/services/flaskapp/src flaskapp
{% endhighlight %}

This will run a container named `flaskapp` based on the image we created in Step 2. The `-v` flag will mount your local source directory inside the container, so your local changes will be reflected automatically.

### **4. Build nginx image**

Given that nginx is a stable piece of software and I probably don't need to muck around with a running nginx container, I just used the official Debian-based nginx image. We'll crete a `server` block in the nginx config to proxy through to our flask app. Create a new directory (doesn't matter where) and create the following `Dockerfile`:

{% highlight ruby %}
FROM nginx:1.7.8
COPY flaskapp.conf /etc/nginx/conf.d/flaskapp.conf
{% endhighlight %}

Now, create a file called flaskapp.conf in the same dir:

{% highlight ruby %}
server {
    listen 80;

    location / {
        proxy_pass http://flaskapp:5000;
    }
}
{% endhighlight %}

Now, build the nginx image as you did for the flask app:

{% highlight bash %}
docker build -t nginx-flask .
{% endhighlight %}

### **5. Run nginx container**

{% highlight bash %}
docker run --name nginx-flask --link flaskapp:flaskapp -v -d -p 8080:80 nginx-flask
{% endhighlight %}

The `--link` flag links the nginx container to the app container and exposes it as a host named with the link alias i.e `flaskapp`. The `-p` flag exposes port 80 of the container on port 8080 of your localhost (or *boot2docker* VM)


That's it, we're done! To access your app, find your boot2docker VM IP using `boot2docker ip` and navigate to `http://$IP:8080` in your favorite browser.

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
