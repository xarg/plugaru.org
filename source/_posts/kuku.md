---
title: 'Introducing kuku: kubernetes templates tool'
date: 2018-10-01 14:54:02
tags: [helm, k8s]
---
At [Gorgias](https://gorgias.io/) we're using [k8s](https://kubernetes.io/) on [gke](https://cloud.google.com/kubernetes-engine/) to run all our production services. We run our REST API apps, RabbitMQ, Celery background workers, PostgreSQL and other smaller services on k8s. We also have staging & development k8s clusters where we experiment with different infrastructure setups and deployments.

If you have multiple k8s clusters to manage, chances are you also need a templating tool to customize your k8s manifests. By far the most popular one is [helm](https://helm.sh) these days. There is also [ksonnet](https://ksonnet.io/) and more recently [pulumi](https://github.com/pulumi/pulumi). All of these tools are powerful and solve real problems, but they are not quite right for us.

I can't speak much about `ksonnet` and `pulumi` because I only briefly had a look at their APIs and how-to guides so take it with a grain of salt. However, as a user, I can speak about `helm` which is what we've been using at Gorgias for all our services.

### Why not helm?

Well, there are a few things I find problematic with helm:

- Poor templating language: requires constant referral to the docs, whitespace issues, yaml formatting is hard.
- Server-side dependency: if you upgrade the server -> every user needs to update their client - waste of valuable time.
- Lack of local validation: helm lint does not actually ensure the validity (Ex: required keys for a k8s object) of the manifest.
- Chart names, releases, and other helm specific features do not fit with our current workflow.

At Gorgias, we've been using Helm to manage all our k8s resources, and it's been great until we had to deal with more complex charts with lots of control flow and `ranges`. If you ever dealt with `ranges` in Helm and `template` you might know that it's not easy to manage considering different contexts. For example `template "name" .` vs `template "name" $` comes to mind. 

### So why not Ksonnet then?

Ksonnet improves the situation a bit with a stricter approach using the [jsonnet](http://jsonnet.org/) language. When I say strict, I mean it doesn't blindly render a text file into YAML as helm does, but uses a real programming language to render the yaml in the end.

My main issue with it is the language: `jsonnet`. It mostly has to do with the fact that it is yet another template language that I have to learn and deal with its different gotchas. A separate issue is that it introduces a whole set of new concepts such as `Part`, `Prototype`, `Parameter`, etc... I found that a bit too much when all I want is to render a bunch of YAML files with some variables.

### Pulumi?

[Pulumi](https://github.com/pulumi/pulumi) approaches the most to what I would consider the ideal tool for us. It uses a programmatic approach where it connects directly to your cluster and creates the resources declared with code (TypeScript, Python, etc..). You write TS code, and you provision your infra with progress-bar. There is a lot to like about this approach. There are, however, a few things that I don't like about Pulumi either: the primary language seems to be TypeScript at the moment - which I don't want to use when it comes to infrastructure code. Python templates were in development when I wrote this post but I didn't try them.

Pulumi also does infrastructure provisioning (multi-provider) - a la Terraform. I think this is overkill for what we need at Gorgias. We don't have to use those features of course, but it seems like it tries to solve 2 different and complex problems at the same time. To put it plainly: it's too much of a swiss army knife for us.

### kuku: a simple templating tool

Finally, after searching for the right tool, I decided that I would write my own. kuku is very similar to Helm, but uses python files as templates instead of YAML. It also doesn't have a server-side component.

Here are some of its goals: 

#### Write python code to generate k8s manifests.
Python is a popular language with a vast ecosystem of dev-ops packages. Most importantly it's easier to debug than some templating languages used today to generate k8s manifests.

#### No k8s server-side dependencies (i.e. tiller).
k8s already has a database for its current state (using etcd). We can connect directly to it (if needed) from the client to do our operations instead of relying on an extra server-side dependency.

#### Local validation of manifests.
Where possible do the validation locally using the [official k8s python client](https://github.com/kubernetes-client/python).

#### Use standard tools.
Where possible use `kubectl` to apply changes to the k8s cluster instead of implementing a specific protocol. Again, this allows for easier maintenance and debugging for the end user.

#### More on Helm comparison

Compared to Helm there is no concept of charts, releases or dependencies. I found that we have rarely used any of those concepts and they just added extra complexity to our charts without much benefit. 

> Instead there are just 2 concepts that are similar to helm: values and templates.

Values come from the CLI or value files (same as Helm). Templates are just python files that have a `template` function.

### Using kuku 

Suppose you want to create a k8s service using a template where you define the service `name`, `internalPort` and `externalPort`.

> To install: `pip3 install kuku`

Given the following `service.py` template:

```python
from kubernetes import client

def template(context):
    return client.V1Service(
        api_version="v1",
        kind="Service",
        metadata=client.V1ObjectMeta(name=context["name"]),
        spec=client.V1ServiceSpec(
            type="NodePort",
            ports=[
                {"port": context["externalPort"], "targetPort": context["internalPort"]}
            ],
            selector={"app": context["name"]},
        ),
    )
```

You can now generate a yaml output from the above template using `kuku` by running: 

```bash
$ ls .
service.py 
$ kuku render -s name=kuku-web,internalPort=80,externalPort=80 .
```

the above produces:

```yaml
# Source: service.py
apiVersion: v1
kind: Service
metadata:
  name: kuku-web
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: kuku-web
  type: NodePort
```
      
You can also combine the above with `kubectl apply -f -` to create your service on k8s:

```bash
kuku render -s name=kuku-web,internalPort=80,externalPort=80 . | kubectl apply -f -
```
    
Same as above, but let's make it shorter:

```bash
kuku apply -s name=kuku-web,internalPort=80,externalPort=80 .
```
   
Finally to delete it: 

```bash
kuku delete -s name=kuku-web,internalPort=80,externalPort=80 .
# same as above
kuku render -s name=kuku-web,internalPort=80,externalPort=80 . | kubectl delete -f - 
```

### kuku templates      

Let's return to templates a bit because a few things are happening there. Templates are python files that are defining a function called `template` that accepts a `dict` argument `context` and returns a k8s object or a list of k8s objects. Simplest example:

```python
def template(context):
    return V1Namespace(name=context['namespace'])  # example k8s object 
```

You can create multiple template files each defining their own `template` function. `kuku` uses the k8s objects (aka models) from [official kubernetes python client package](https://github.com/kubernetes-client/python). You can find them all [here](https://github.com/kubernetes-client/python/blob/master/kubernetes/README.md#documentation-for-models).

When writing kuku templates I highly recommend that you use an editor that is aware of the k8s python package above so you can get nice auto-completion of properties - it makes life some much easier as a result.

### kuku command line interface 

Similar to [helm](https://helm.sh/), `kuku` accepts defining it's context variables from the CLI:

```bash
kuku render -s namespace=kuku .
```
    
`-s namespace=kuku` will be passed to the `context` argument in your `template` function. Run `kuku -h` to find out more.


### A more realistic example

Defining services and a namespace is nice, but let's see how it behaves with a more complex Postgres StatefulSet. 
Consider the following directory:

```
.
├── templates
│   ├── configmap.py
│   ├── service.py
│   └── statefulset.py
├── values-production.yaml
└── values-staging.yaml
```

We have some value files, a configmap, service (like before) and statefulset template. This postgres statefulset template is something similar to what we have currently in our production at Gorgias.

Let's have a look at `values-production.yaml`:

```yaml
name: pg # global name of our statefulset/service/configmap/etc..

image: postgres:latest

# optional
nodeSelector:
  cloud.google.com/gke-nodepool: pg

replicas: 1

resources:
  requests:
    memory: 10Gi
  limits:
    memory: 12Gi
    
pvc:
- name: data
  class: ssd
  size: 500Gi
  mountPath: /var/lib/postgresql/data/

configmap:
- name: postgresql.conf
  value: |
    max_connections = 500
```

Above we're defining values that are used to declare that we want to run one instance of `postgres:latest` docker image on a specific k8s node pool while requesting some memory and a persistent volume. We're also using a config map to define our `postgresql.conf` so it's easier to keep track of its changes.

Keep in mind the above values and now let's have a look at our `statefuset.py` template:

```python
from kubernetes import client


def template(context):
    # volumes attached to our pod
    pod_spec_volumes = []
    
    # where those volumes are mounted in our container
    pod_spec_volume_mounts = []
    
    # persistent volume claims templates
    stateful_set_spec_volume_claim_templates = []

    # only set the claims if we have a PVC value
    for pvc in context.get("pvc"):
        stateful_set_spec_volume_claim_templates.append(
            client.V1PersistentVolumeClaim(
                metadata=client.V1ObjectMeta(
                    name=pvc["name"],
                    annotations={
                        "volume.beta.kubernetes.io/storage-class": pvc["class"]
                    },
                ),
                spec=client.V1PersistentVolumeClaimSpec(
                    access_modes=["ReadWriteOnce"],
                    resources=client.V1ResourceRequirements(
                        requests={"storage": pvc["size"]}
                    ),
                ),
            )
        )
        pod_spec_volume_mounts.append(
            client.V1VolumeMount(name=pvc["name"], mount_path=pvc["mountPath"])
        )

    # same for configmap
    if "configmap" in context:
        volume_name = "{}-config".format(context["name"])
        pod_spec_volumes.append(
            client.V1Volume(name=volume_name, config_map=context["name"])
        )
        pod_spec_volume_mounts.append(
            client.V1VolumeMount(name=volume_name, mount_path="/etc/postgresql/")
        )

    # command to check if postgres is live (used for probes below)
    pg_isready_exec = client.V1ExecAction(command=["gosu postgres pg_isready"])

    return client.V1StatefulSet(
        api_version="apps/v1beta1",
        kind="StatefulSet",
        metadata=client.V1ObjectMeta(name=context["name"]),
        spec=client.V1StatefulSetSpec(
            service_name=context["name"],
            replicas=context["replicas"],
            selector={"app": context["name"]},
            template=client.V1PodTemplateSpec(
                metadata=client.V1ObjectMeta(labels={"name": context["name"]}),
                spec=client.V1PodSpec(
                    containers=[
                        client.V1Container(
                            name="postgres",
                            image=context["image"],
                            lifecycle=client.V1Lifecycle(
                                pre_stop=client.V1Handler(
                                    _exec=client.V1ExecAction(
                                        command=[
                                            'gosu postgres pg_ctl -D "$PGDATA" -m fast -w stop'
                                        ]
                                    )
                                )
                            ),
                            liveness_probe=client.V1Probe(
                                _exec=pg_isready_exec,
                                initial_delay_seconds=120,
                                timeout_seconds=5,
                                failure_threshold=6,
                            ),
                            readiness_probe=client.V1Probe(
                                _exec=pg_isready_exec,
                                initial_delay_seconds=10,
                                timeout_seconds=5,
                                period_seconds=30,
                                failure_threshold=999,
                            ),
                            ports=[client.V1ContainerPort(container_port=5432)],
                            volume_mounts=pod_spec_volume_mounts,
                            resources=client.V1ResourceRequirements(
                                **context["resources"]
                            )
                            if "resources" in context
                            else None,
                        )
                    ],
                    volumes=pod_spec_volumes,
                    node_selector=context.get("nodeSelector"),
                ),
            ),
            volume_claim_templates=stateful_set_spec_volume_claim_templates,
        ),
    )
```

If you squint a bit you might see that the last return is similar to a yaml file, but it uses python objects instead with all of it's IFs and for-loop, standard library, etc..

What I find better than a regular helm YAML template is that you can validate some of the input arguments of those python objects (Ex: `client.V1Container`) even before the template is sent to your k8s server - not to mention autocomplete. 

Finally, this is how it all comes together:

```bash
kuku render -f values-production.yaml templates/ | kubectl apply -f -
```

The above renders all your templates and generates the yaml manifests that are then applied using `kubectl apply`.

You can find the source here: https://github.com/xarg/kuku/
And a couple of examples: https://github.com/xarg/kuku/tree/master/examples

## Conclusion

We've started using kuku in Sept. 2018 at [Gorgias](https://gorgias.io), and we've since migrated all our Helm charts to kuku templates. It allowed us to customize our k8s deployment code to our needs much easier than before with minimal deployment surprises.

Hope you find kuku useful as we did. Happy hacking!
