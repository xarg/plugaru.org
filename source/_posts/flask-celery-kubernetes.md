---
title: Running Flask & Celery with Kubernetes
date: 2016-03-03 16:49:31
categories: devops 
tags: [kubernetes, k8s, flask, celery, python]
hidden: true
---

At [Gorgias](https://gorgias.io) we recently switched our [flask](http://flask.pocoo.org) & [celery](http://www.celeryproject.org/) apps from Google Cloud VMs provisioned with [Fabric](http://www.fabfile.org/) to using [docker](https://www.docker.com/) with [kubernetes](http://kubernetes.io/) (k8s). This is a post about our experience doing this.

**Note:** I'm assuming that you're somewhat familiar with Docker.

## Docker structure

The killer feature of Docker for us is that it allows us to make layered binary images of our app. What this means is that you can start with a minimal base image, then make a python image on top of that, then an app image on top of the python one, etc..

Here's the hierarchy of our docker images:

* [gorgias/pgbouncer](https://github.com/mbentley/dockerfiles/tree/master/ubuntu/pgbouncer)
* [gorgias/rabbitmq](https://gist.github.com/xarg/7fe2af07ebd7a4b9bdca)
* gorgias/nginx *- extends gorgias/base and installs NGINX*
* gorgias/python3 *- Installs pip, python3.5 - yes, using it in production.*
 * gorgias/app *- This installs all the system dependencies: libpq, libxml, etc.. and then does pip install -r requirements.txt*
     * gorgias/web *- this sets up uWSGI and runs our flask app*
     * gorgias/worker *- Celery worker*

**Piece of advice:** If you used to run your app using supervisord before I would advise to avoid the temptation to do the same with docker, just let your container crash and let your Kubernetes/Swarm/Mesos handle it.

Now we can run the above images using: [docker-compose](https://docs.docker.com/compose/), [docker-swarm](https://docs.docker.com/swarm/overview/), [k8s](http://kubernetes.io/), [Mesos](https://mesosphere.com/), etc...

<!-- more -->

## We chose Kubernetes (k8s) too
There is an excellent [post](http://code.haleby.se/2016/02/12/why-we-chose-kubernetes/) about the differences between container deployments which also settles for k8s.

I'll also just assume that you already did your homework and you plan to use k8s. But just to put more data out there:

> **Main reason:** We are using [Google Cloud](https://cloud.google.com/) already and it provides a ready to use Kubernetes cluster on their cloud.

This is huge as we don't have to manage the k8s cluster and can focus on deploying our apps to production instead.

Let's begin by making a list of what we need to run our app in production:

* Database (Postgres)
* Message queue (RabbitMQ)
* App servers (uWSGI running Flask)
* Web servers (NGINX proxies uWSGI and serves static files)
* Workers (celery)

### Why Kubernetes again?
We ran the above in a normal VM environment, why would we need k8s? To understand this, let's dig a bit into what k8s offers:

* A **pod** is a group of containers (docker, rtk, lxc...) that runs on a **Node**. It's a group because sometimes you want to run a few containers next to each other. For example we are running uWSGI and NGINX on the same pod (on the same VM and they share the same ip, ports, etc..).
* A **Node** is a machine (VM or metal) that runs a k8s daemon (minion) that runs the Pods.  
* The nodes are managed by the k8s master (which in our case is managed by the container engine from Google).
* **Replication Controller** or for short **rc** tells k8s how many pods of a certain type to run. Note that you don't tell k8s where to run them, it's master's job to schedule them. They are also used to do rolling updates, and autoscaling. Pure awesome.
* **Services** take the exposed ports of your Pods and publishes them (usually to the Public). Now what's cool about a service that it can load-balance the connections to your pods, so you don't need to manage your HAProxy or NGINX. It uses labels to figure out what pods to include in it's pool.
* **Labels**: The CSS selectors of k8s - use them everywhere!

There are more concepts like volumes, claims, secrets, but let's not worry about them for now.

### Postgres

We're using Postgres as our main storage and we are ~~**not running it using Kubernetes**~~.

**Now we are running postgres in k8s (1 master + 1 hot standby + wal-e), you can ignore the rest of this paragraph.**

The reason here is that we wanted to run Postgres using provisioned SSD + high memory instances. We could have created a cluster just for postgres with these types of machines, but it seemed like an overkill.

The philosophy of k8s is that you should design your cluster with the thought that pods/nodes of your cluster are just gonna die randomly. I haven't figured our how to setup Postgres with this constraint in mind. So we're just running it replicated with a hot-standby and doing backups with [wall-e](https://github.com/wal-e/wal-e) for now. If you want to try it with k8s there is a guide [here](https://blog.oestrich.org/2015/08/running-postgres-inside-kubernetes/). And make sure you tell us about it.

### RabbitMQ
RabbitMQ (used as message broker for Celery) is running on k8s as it's easier (than Postgres) to make a cluster. Not gonna dive into the details. It's using a replication controller to run 3 pods containing rabbitmq instances. This guide helped: https://www.rabbitmq.com/clustering.html.

### uWSGI & NGINX
As I mentioned before, we're using a replication controller to run 3 pods, each containing uWSGI & NGINX containers duo: gorgias/web & gorgias/nginx. Here's our replication controller `web-rc.yaml` config:

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: web
spec:
  replicas: 3 # how many copies of the template below we need to run
  selector:
    app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: gcr.io/your-project/web:latest # the image that you pushed to Google Container Registry using gcloud docker push
        ports: # these are the exposed ports of your Pods that are later used by the k8s Service
          - containerPort: 3033 
            name: "uwsgi"
          - containerPort: 9099
            name: "stats"
      - name: nginx
        image: gcr.io/your-project/nginx:latest
        ports:
          - containerPort: 8000
            name: "http"
          - containerPort: 4430
            name: "https"
        volumeMounts: # this holds our SSL keys to be used with nginx. I haven't found a way to use the http load balancer of google with k8s.  
          - name: "secrets"
            mountPath: "/path/to/secrets"
            readOnly: true
      volumes:
        - name: "secrets"
          secret:
            secretName: "ssl-secret"
```

And now the `web-service.yaml`:

```
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  ports:
  - port: 80
    targetPort: 8000
    name: "http"
    protocol: TCP
  - port: 443
    targetPort: 4430
    name: "https"
    protocol: TCP
  selector:
    app: web
  type: LoadBalancer
```

That `type: LoadBalancer` at the end is super important because it tells k8s to request a public IP and route the network to the Pods with the selector=app:web.
If you're doing a rolling-update or just restarting your pods, you don't have to change the service. It will look for pods matching those labels.

### Celery

Also a replication controller that runs 4 pods containing a single container: `gorgias/worker`, but doesn't need a service as it only consumes stuff. Here's our `worker-rc.yaml`:

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: worker
spec:
  replicas: 2
  selector:
    app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      containers:
      - name: worker
        image: gcr.io/your-project/worker:latest
```

#### Some tips

- Installing some python deps take a long time, for stuff like numpy, scipy, etc.. try to install them in your `namespace/app` container using pip and then do another pip install in the container that extends it, ex: `namespace/web`, this way you don't have to rebuild all the deps every time you update one package or just update your app.
- Spend some time playing with [gcloud](https://cloud.google.com/sdk/) and [kubectl](https://cloud.google.com/container-engine/docs/kubectl/). This will be the fastest way to learn of google cloud and k8s.
- Base image choice is important. I tried [phusion/baseimage](https://github.com/phusion/baseimage-docker) and [ubuntu/core](https://wiki.ubuntu.com/Core). Settled for `phusion/baseimage` because it seems to handle the init part better than ubuntu core. They still feel too heavy. `phusion/baseimage` is 188MB.

#### Conclusion

With Kubernetes, docker finally started to make sense to me. It's great because it provides great tools out of the box for doing web app deployment. Replication controllers, Services (with LoadBalancer included), Persistent Volumes,  internal DNS. It should have all you need to make a resilient web app fast.

At [Gorgias](https://gorgias.io) we're building a next generation helpdesk that allows responding 2x faster to common customer requests and having a fast and reliable infrastructure is crucial to achieve our goals. 

If you're interested in working with this kind of stuff (especially to improve it): [we're hiring](https://gorgias.workable.com)!
