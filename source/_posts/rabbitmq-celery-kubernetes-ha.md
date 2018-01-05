---
title: Highly available Celery with RabbitMQ and Kubernetes
date: 2018-01-02 16:52:03
categories: devops 
tags: [kubernetes, helm, rabbitmq, celery, ha]
---

[Kubernetes](https://kubernetes.io/), [RabbitMQ](https://www.rabbitmq.com/) and [Celery](http://www.celeryproject.org/) provides a very natural way to create a reliable python worker cluster. This post is based on my experience running Celery in production at [Gorgias](https://gorgias.io) over the past 3 years. The scope of this post is mostly dev-ops setup and a few small gotchas that could prove useful for people trying to accomplish the same type of deployment. At the end we'll try to shut down a machine to see if our cluster is indeed reliable as I claim.

Before diving too deep I recommend refreshing your knowledge on the tools we are going to use. In particular I expect:

- You have some familiarity with Kubernetes and in particular: pods, services, deployments, stateful sets and persistent volumes.
- You used or know a bit about RabbitMQ. Some basics about clustering would be nice, but not required. You can read their docs here: https://www.rabbitmq.com/clustering.html
- You used or know about Celery: how it schedules it's tasks and executes them.

### Pieces falling into place: Why Kubernetes, RabbitMQ and Celery?

- **Kubernetes** is a very reliable container orchestration system (runs your docker images). It is used by a vast number of companies in production environments. It's proven tech and provides a good base for making failure tollerant and scalable applications such as an async worker cluster which is what we're trying to do.
- **RabbitMQ** is a popular open source broker that has a history of being resilient to failure, can be configured to be highly-available and can protect your environment from data-loss in case of a hardware failure.
- **Celery** is probably the most popular python async worker at this moment. It's feature rich, stable and actively maintained. Celery (or any other worker) by it's nature is distributed and relies on the message broker (RabbitMQ in our case) for state synchronisation. It's also what we use at [Gorgias](https://gorgias.io) to run asynchronous tasks.

Given their properties I hope that when we put all of the above components together you will have a robust worker cluster that is easy to scale and will be hard to brake.

### Kubernetes (k8s) and Helm setup

First we'll need to run k8s either via [minikube](https://github.com/kubernetes/minikube) on your local machine or using some k8s provider such as [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/) (GKE). For this tutorial I'm going to use GKE, but feel free to use any kubernetes environment.

Assuming you have your [gcloud](https://cloud.google.com/sdk/docs/) setup let's create a new 3 node Kubernetes cluster. It will take a few minutes so feel free to grab some refreshment:

    gcloud container clusters create ha-celery --num-nodes=3 -z us-east1-c

Make sure you delete your cluster (so you don't get charged) after you're done like so:

    gcloud container clusters delete ha-celery -z us-east1-c
    
Test that you have access to it:

    kubectl cluster-info

Great. Now let's install [Helm](https://github.com/kubernetes/helm). Helm is a package manager that will allow us to
create a template for our RabbitMQ and Celery deployment. We can use it to run a development, staging or production environment without having to maintain separate configs for each one. It's also easier to use than running `kubectl` commands when dealing with multiple k8s primitives at a time.

To install helm in your kubernetes cluster (make sure the [Helm CLI](https://docs.helm.sh/using_helm/#installing-helm) is installed on your machine first):

    helm init
    
At this point you should be all set with Kubernetes and Helm. We are now ready to deploy our RabbitMQ cluster which Celery will use later on.

### RabbitMQ (RMQ) docker image 

In order to run our RabbitMQ (RMQ) cluster on k8s first we'll have to build the Docker images for it. Here's a sample Dockerfile:

```
FROM rabbitmq:3.6.12

RUN rabbitmq-plugins enable --offline rabbitmq_management

ENV RABBITMQ_ERLANG_COOKIE changeThis

# Add files.
COPY ./cmd.sh /
COPY ./rabbitmq.config /etc/rabbitmq/rabbitmq.config

# Define default command.
CMD ["/cmd.sh"]

# default ports + management plugin ports
EXPOSE 4369 5671 5672 25672 15671 15672
```

Above we have a custom RabbitMQ Dockerfile that inherits the official RabbitMQ image and adds a few extra features.
It installs the `rabbitmq_management` plugin which I highly recommend if you want to understand what's going on in production. I copies our `rabbitmq.config` and `cmd.sh` which will see later. And finally exposes the default RMQ ports.

Let's have a look at `rabbitmq.config`:

```
[
    { rabbit, [
            { loopback_users, [ ] },
            { tcp_listeners, [ 5672 ] },
            { ssl_listeners, [ ] },
            { hipe_compile, false },
            { cluster_partition_handling, pause_minority}
    ] },
    { rabbitmq_management, [ { listener, [
            { port, 15672 },
            { ssl, false }
    ] } ] }
].
```

The first part about `loopback_users` and listeners is pretty straight forward. `hipe_compile` setting is false, but can be true if you use the high-perf Erlang.

Now what I believe to be an extremely important setting is [cluster_partition_handling](https://www.rabbitmq.com/partitions.html#pause-minority) which has the default value set to `ignore`. It's extremely important for it to be `pause_minority` instead in order to avoid [split-world](https://en.wikipedia.org/wiki/Split-brain_%28computing%29) and data loss situations. Here's why:

Imagine you have a RMQ cluster of 3 nodes setup: `rmq0`, `rmq1` and `rmq2`.
All is running smoothly and then suddenly you have a [network partition](https://en.wikipedia.org/wiki/Network_partition) that separates `rmq0` from the other 2 nodes. Note that all the nodes are still running, it's just that they can't communicate between themselves.
Now if you have `cluster_partition_handling` set to the default `ignore` all the clients connected to `rmq0` will still be able to read from it and most importantly write to it! Here's a more concrete example:

Suppose we have 2 Celery workers (`w0` and `w1`) that get tasks from the `celery` queue (the default one for celery).
`w0` is connected to `rmq0` and consumes the `celery` queue and `w1` that is connected to `rmq1` and does the same.

In the case of a network partition with the RMQ defaults `w0` will consume `celery` as if nothing happened, the problem is that `w1` also does the same on `rmq1`. In this situation the same task can be consumed 2 times! When the network partition is fixed which of the `rmq` nodes in your cluster holds the truth?
This is called a split-world or split-brain situation and it literally can cause your brain to split trying to untangle the mess. This happens more often than you might think even on very reliable hardware and network. In this situation you're better off re-creating the queue from scratch with some nasty data loss in the process.

Basically what `cluster_partition_handling` set to `ignore` is saying: In the case of a network partition the RMQ nodes will just ignore this failure and continue running as if nothing happened accepting regular consumer operations. Setting it to `pause_minority` will cause the nodes in the minority (in our case `rmq-0`) to pause - becoming read-only basically. Once the cluster is back it should sync with the other nodes and get back on track. This IMHO is what the default should be because it avoids the split-world situations which is what I believe most really want.

Ok and finally the `cmd.sh`:

```
#!/usr/bin/env bash

ulimit -n 65536

chown -R rabbitmq:rabbitmq /var/lib/rabbitmq

# This is needed for statefulset service name resolution - needed for short name resolution
if [ -n "$RABBITMQ_SERVICE_DOMAIN" ]; then
    echo "search $RABBITMQ_SERVICE_DOMAIN" >> /etc/resolv.conf
fi

is_clustered="/var/lib/rabbitmq/is_clustered"

host=`hostname`

join_cluster () {
    if [ -e $is_clustered ]; then
        echo "Already clustered with $CLUSTER_WITH"
    else
        # Don't cluster with self or if already clustered
        if ! [[ $CLUSTER_WITH =~ $host ]]; then
            rabbitmq-server -detached
            rabbitmqctl stop_app
            rabbitmqctl join_cluster rabbit@$CLUSTER_WITH
            rabbitmqctl start_app

            # mark that this node is clustered
            mkdir $is_clustered
            # stopping because it we started it later in attached mode
            rabbitmqctl stop
            sleep 5
        fi
    fi
}

create_vhost() {
    rabbitmq-server -detached
    until rabbitmqctl node_health_check; do echo "Waiting to start..." && sleep 1; done;

    USER_EXISTS=`rabbitmqctl list_users | { grep $RABBITMQ_USERNAME || true; }`

    # create user only if it doesn't exist
    if [ -z "$USER_EXISTS" ]; then
        rabbitmqctl add_user $RABBITMQ_USERNAME $RABBITMQ_PASSWORD
        rabbitmqctl add_vhost $RABBITMQ_VHOST
        rabbitmqctl set_permissions -p $RABBITMQ_VHOST $RABBITMQ_USERNAME ".*" ".*" ".*"
        rabbitmqctl set_policy -p $RABBITMQ_VHOST ha-all "" '{"ha-mode":"all","ha-sync-mode":"automatic"}'
    fi
    # stopping because it we started it later in attached mode
    rabbitmqctl stop
    sleep 5
}

if [ -n "$CLUSTERED" ] && [ -n "$CLUSTER_WITH" ]; then
    join_cluster
fi

if [ -n "$RABBITMQ_USERNAME" -a -n "$RABBITMQ_PASSWORD" -a -n "$RABBITMQ_VHOST" ]; then
    create_vhost
fi

rabbitmq-server $RABBITMQ_SERVER_PARAMS
```

The above script will attempt to join a cluster if the `$CLUSTERED` and `$CLUSTER_WITH` vars are set and then attempt to create a RMQ virtual host and a user. This part below sets the default HA policy for the vhost which in our case here is `all` meaning that all queues will be replicated across all nodes:

    rabbitmqctl set_policy -p $RABBITMQ_VHOST ha-all "" '{"ha-mode":"all","ha-sync-mode":"automatic"}'
    
Also note the `$RABBITMQ_SERVICE_DOMAIN` variable. That one is needed in order to have our nodes to connect to each other. It's value is the service domain of the k8s service we're going to use.

Now that we understand our docker image better we can build it and upload it to our gcr.io docker registry.

    git clone git@github.com:xarg/rabbitmq-statefulset.git
    cd rabbitmq-statefulset
    docker build -t gcr.io/your-project/rabbitmq .
    gcloud docker -- push gcr.io/your-project/rabbitmq
    
We have our image uploaded to the container registry and now we're ready to deploy it!
    
### RabbitMQ Helm chart 

To deploy our RabbitMQ cluster we're going to use the kubernetes [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) to deploy our RMQ cluster. The reason for this choice is that StatefulSets is in it's description:

> A StatefulSet [..] Manages the deployment and scaling of a set of Pods, and provides guarantees about the ordering and uniqueness of these Pods.

This is important when doing upgrades of the cluster for example, but also because the hostnames of the rabbitmq nodes need to be kept the same between restarts. See here: https://www.rabbitmq.com/clustering.html#issues-hostname

Let's have a look at our helm chart files:

```
.
└── rabbitmq
    ├── Chart.yaml -- metadata
    ├── templates
    │   ├── _helpers.tpl -- helper functions
    │   ├── service.yaml -- template for the k8s service
    │   ├── ssd.yaml -- ssd storage (if IO is important)
    │   ├── standard.yaml -- standard storage (default).
    │   └── statefulset.yaml -- template for the statefulset
    └── values.yaml -- default values for our templates 
```

A helm chart is basically just a template that uses the values from the `values.yaml` to render the template.
We'll have a look at 2 files

`service.yaml`:
```
apiVersion: v1
kind: Service
metadata:
  name: rmq
spec:
  ports:
    ...
  clusterIP: None
  selector:
    app: rmq
```

This service defines a way for our application and the cluster to connect to their nodes. Note that the `clusterIP: None` - that's because we don't want k8s to load-balance our RMQ cluster, we'll do that on the Celery application level.

`statefulset.yaml`:
```
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: rmq
spec:
  serviceName: rmq
  replicas: {{ .Values.replicaCount }}
  
  volumeClaimTemplates:
  - metadata:
      name: rmq
      annotations:
        volume.beta.kubernetes.io/storage-class: "standard"
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
          
  template:
    metadata:
      labels:
        app: rmq
    spec:
      # prevents rmq pods running on the same nodes                             
      affinity:                                                                 
        podAntiAffinity:                                                        
          preferredDuringSchedulingIgnoredDuringExecution:                      
          - weight: 100                                                         
            podAffinityTerm:                                                    
              labelSelector:                                                    
                matchExpressions:                                               
                - key: app                                                      
                  operator: In                                                  
                  values:                                                       
                  - rmq                                                         
              topologyKey: kubernetes.io/hostname   
      containers:
      - name: rmq
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        volumeMounts:
          - mountPath: /var/lib/rabbitmq
            name: rmq
        env:
          - name: CLUSTERED
            value: "true"
          - name: CLUSTER_WITH
            value: "rmq-0"
          - name: RABBITMQ_SERVICE_DOMAIN
            value: "rmq.default.svc.cluster.local"
{{ if eq .Values.environment "production" }}
        resources:
{{ toYaml .Values.resources | indent 10 }}
{{ end }}
{{ if .Values.nodeSelector }}
      nodeSelector:
{{ toYaml .Values.nodeSelector | indent 8 }}
{{ end }}
```

The `StatefulSet` will first claim a `10Gi` persistent volume with the `standard` storage type for each of our nodes defined by `replicaCount`. Then it will start each rabbitmq pod and check if it's alive and running.

### Deploy the RabbitMQ cluster in Kubernetes

While still in the `rabbitmq-statefulset` directory we can deploy our chart to k8s using the helm command:

    helm upgrade -i rmq --set environment=production --set image.repository=gcr.io/your-project/rabbitmq charts/rabbitmq/

If everything went well you should eventually see:

```
kubectl get pod
NAME      READY     STATUS    RESTARTS   AGE
rmq-0     1/1       Running   0          4m
rmq-1     1/1       Running   0          3m
rmq-2     1/1       Running   0          2m
```

 You now have a working RMQ cluster with 3 running nodes. I recommend poking around a bit. Check their logs with `kubectl logs rmq-0`. Check the created service: `kubectl get svc`, etc..

Now that our cluster is created and running we can finally start working on the workers.

## Celery application

Let's create a simple celery application with workers that just wait 10 seconds. We'll also create a script that schedules a lot of celery tasks.

`tasks.py`:
```
import time
from celery import Celery

BROKER_URL = [
    'amqp://my_user:my_pass@rmq-0.rmq.default.svc.cluster.local:5672/my_vhost',
    'amqp://my_user:my_pass@rmq-1.rmq.default.svc.cluster.local:5672/my_vhost',
    'amqp://my_user:my_pass@rmq-2.rmq.default.svc.cluster.local:5672/my_vhost',
]
app = Celery('tasks', broker=BROKER_URL)


@app.task
def count():
    for i in range(10):
        print(i)
        time.sleep(1)

```

As you can see all this task does is waits 10s and prints it's counts.

`scheduler.py`:
```
from tasks import count

# just schedules 10k count tasks
for i in range(10000):
    count.delay()

```

The scheduler will be used to schedule lots of count tasks so we can observe their execution while killing workers, rabbitmq nodes, etc..

Now that we have our handler and scheduler we can put then in a simple Docker image so we can run them in k8s:

```
FROM python:3

RUN pip install celery 

COPY src/* /

CMD ["/usr/local/bin/celery", "-A", "tasks", "worker", "--loglevel=info"]

``` 

You can find the source for the handler, scheduler, Dockerfile and the Helm chart here: https://github.com/xarg/celery-counter

Now let's build the image and push to the registry just like we did with the RMQ image:

    git clone git@github.com:xarg/celery-counter.git
    cd celery-counter
    docker build -t gcr.io/your-project/celery-counter .                            
    gcloud docker -- push gcr.io/your-project/celery-counter                        
                                                                                
Great, we're getting closer to running our counter worker tasks on k8s.

    
### Celery Helm chart

Now that we have our image uploaded we can deploy it once again using the celery-counter helm chart. While still in the `celery-counter` directory:

    helm upgrade -i counter --set image.repository=gcr.io/your-project/celery-counter charts/counter
    
I not going to dig into this helm chart: It's a simple k8s [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) with 3 replicas that runs the celery worker command.
Make sure that we workers are running using: `kubectl get pod` and then look at their logs using `kubectl logs <name of the pod>`

### Schedule our tasks

Hopefully you have your workers running by now, but they are not doing anything yet. We need to use our scheduler to make then execute the count tasks:

    kubectl create -f scheduler.yaml
    
Once the scheduler is started you should start seeing some activity in your worker logs:

```
[INFO/ForkPoolWorker-1] Task tasks.count[cf168fdb-c199-481b-81cc-7ac30c93786e] succeeded in 10.015281924999726s: None
[WARNING/ForkPoolWorker-1] 0
[WARNING/ForkPoolWorker-1] 1
[WARNING/ForkPoolWorker-1] 2
[WARNING/ForkPoolWorker-1] 3
[WARNING/ForkPoolWorker-1] 4
[WARNING/ForkPoolWorker-1] 5
[WARNING/ForkPoolWorker-1] 6
[WARNING/ForkPoolWorker-1] 7
[WARNING/ForkPoolWorker-1] 8
[WARNING/ForkPoolWorker-1] 9
```

Nice work! You have your worker cluster running and executing tasks!

## Redundancy 

Now that we have our Celery counter application running we can go ahead and remove a k8s node to see what happens.

First let's see on which nodes our pods are running:

```
kubectl get pods -o wide
NAME                               READY     STATUS     NODE
counter-counter-3473528255-24z8m   1/1       Running    gke-ha-celery-default-pool-5939d2ec-khq0
counter-counter-3473528255-6n3q8   1/1       Running    gke-ha-celery-default-pool-5939d2ec-dc85
counter-counter-3473528255-g823p   1/1       Running    gke-ha-celery-default-pool-5939d2ec-dc85
rmq-0                              1/1       Running    gke-ha-celery-default-pool-5939d2ec-dc85
rmq-1                              1/1       Running    gke-ha-celery-default-pool-5939d2ec-khq0
rmq-2                              1/1       Running    gke-ha-celery-default-pool-5939d2ec-43nq
```

Observe on which node `rmq-0` is running. In the above example it's `gke-ha-celery-default-pool-5939d2ec-dc85` - this is the node we're going to remove from the pool to cause a little havoc.

By default celery connects to the first RMQ node in the list (see `BROKER_URL` in the `tasks.py`) and then if the connection fails to the first broker it goes to the next one, etc..

Before killing a k8s node to see what happens let's observe our cluster using these commands (run each one in it's own terminal):

    # same as above - but choose a pod that is NOT running on a node that you plan to kill
    # the reason for this is to observe how the works fail over to a different RMQ node.
    kubectl logs -f counter-counter-3473528255-24z8m
    
    # to see how rmq-0 dies
    kubectl logs -f rmq-0
    
    # to see how rmq-1 takes over the traffic from the workers
    kubectl logs -f rmq-1
    
    # to see how pods stop and then start again
    watch "kubectl get pods -o wide"
    
    # to see how the node gets unscheduled
    watch "kubectl get nodes"
    
Next we'll mark the k8s node that runs `rmq-0` as unschedulable and then we'll drain (kill) all pods on it, you should choose the the node that runs the `rmq-0`, you can see it running this command `kubectl get pods -o wide`:

    # marks the node unschedulable (the one that runs rmq-0)
    kubectl cordon gke-ha-celery-default-pool-5939d2ec-dc85
    
    # kills all pods that run on it
    kubectl drain --force --ignore-daemonsets gke-ha-celery-default-pool-5939d2ec-dc85
    
Now keep your eyes on your monitoring commands. You should see how `rmq-0` is dying and 1 or 2 celery counters since they might be running on the same node as `rmq-0`.
If everything went according to plan you should see a log in your worker like: 

    consumer: Connection to broker lost. Trying to re-establish the connection...   
    ...
    Cannot connect to amqp://my_user:**@rmq-0.rmq.default.svc.cluster.local:5672/my_vhost: [Errno -2] Name or service not known.
    ...
    Connected to amqp://my_user:**@rmq-1.rmq.default.svc.cluster.local:5672/my_vhost

And then just as before continue to execute the tasks after the period of failure.

I also recommend looking at the `rmq-1` logs to see how the clients start connecting to it and then how it accepts again the rmq-0 into the cluster once it gets up again.
    
## Conclusion

What we did so far:

- Create a new k8s cluster
- Build and pushed the RabbitMQ and Celery images to the Google Container Registry
- Deployed the helm charts for RabbitMQ and celery cluster.
- Removed a k8s node and observed the behavior of our workers and RMQ nodes.

Of course this is just one way you cluster can fail. There are many other things that can happen. I recommend looking at: https://github.com/bloomberg/powerfulseal or https://github.com/asobti/kube-monkey to get an idea about other possible types of failure.

If I had to give one piece of advice based on this article:

> Expect your workers to die at any moment and always code with that in mind.

IMHO this is the hardest part of all.

At Gorgias we're sending tons of emails/chats and facebook messages and also making HTTP requests to user defined HTTP endpoints before sending the aforementioned messages which can fail with a timeout or an error. These are just a few questions that arise: 

- Should the emails be sent if other parts of the process have passed or not? If not how should we notify the customer?
- What happens if the HTTP service we're trying to reach times out? How many times should we retry before giving up? What happens then?
- What if the worker is killed in the middle of the transaction with the mail server? Was the email sent or not? Should we retry? Should we notify the customer?
- If the mail server is down do we have a retry mechanism and when should the retry switch to a different server?

The above are but a few of the questions that come to mind when thinking of our application, in reality there are many more and the answer is not always simple. The code required for failure handling makes the application much more verbose, harder to debug and maintain. We're try having less and simpler features precisely because some many things can go wrong. 

Even though the application level failure handling is very hard it's still a million times better when you know that you can rely on Kubernetes and RabbitMQ to stay up and running your application code even if your VMs or physical machines go down. It's a lot easier to build resilient and scalable applications that it was before kubernetes in my option and I hope that this post illustrates just that.

Here are the repos that have been used in this post:

- RabbitMQ StatefulSet Helm chart: https://github.com/xarg/rabbitmq-statefulset
- Celery counter Docker image and Helm chart: https://github.com/xarg/celery-counter