---
title: PostgreSQL backup with pghoard & Kubernetes
date: 2016-07-17 16:50:33
categories: devops 
tags: [kubernetes, k8s, postgres, pghoard]
---

**TLDR:** https://github.com/xarg/pghoard-k8s

This is a small tutorial on how to do incremental backups using pghoard  for your PostgreSQL (I assume you’re running everything in Kubernetes). This is intended to help people to get started faster and not waste time finding the right dependencies, etc..

[pghoard](https://github.com/ohmu/pghoard) is a PostgreSQL backup daemon that incrementally backups your files on a object storage (S3, Google Cloud Storage, etc..).
For this tutorial what we’re trying to achieve is to upload our PostgreSQL to S3. 

First, let’s create our docker image (we’re using the [alpine:3.4](https://hub.docker.com/_/alpine/) image cause it’s small):

```
FROM alpine:3.4

ENV REPLICA_USER "replica"
ENV REPLICA_PASSWORD "replica"

RUN apk add --no-cache \
    bash \
    build-base \        
    python3 \
    python3-dev \
    ca-certificates \
    postgresql \
    postgresql-dev \
    libffi-dev \
    snappy-dev
RUN python3 -m ensurepip && \
    rm -r /usr/lib/python*/ensurepip && \
    pip3 install --upgrade pip setuptools && \
    rm -r /root/.cache && \
    pip3 install boto pghoard 


COPY pghoard.json /pghoard.json.template
COPY pghoard.sh /

CMD /pghoard.sh
```
`REPLICA_USER` and `REPLICA_PASSWORD` env vars will be replaced later in your Kubernetes conf by whatever your config is in production, I use those values to test locally using docker-compose.

The config `pghoard.json` which tells where to get your data from and where to upload it and how:
```
{
    "backup_location": "/data",
    "backup_sites": {
        "default": {
            "active_backup_mode": "pg_receivexlog",
            "basebackup_count": 2,
            "basebackup_interval_hours": 24,
            "nodes": [
                {
                    "host": "YOUR-PG-HOST",
                    "port": 5432,
                    "user": "replica",
                    "password": "replica",
                    "application_name": "pghoard"
                }
            ],
            "object_storage": {
                "aws_access_key_id": "REPLACE",
                "aws_secret_access_key": "REPLACE",
                "bucket_name": "REPLACE",
                "region": "us-east-1",
                "storage_type": "s3"
            },
            "pg_bin_directory": "/usr/bin"
        }
    },
    "http_address": "127.0.0.1",
    "http_port": 16000,
    "log_level": "INFO",
    "syslog": false,
    "syslog_address": "/dev/log",
    "syslog_facility": "local2"
}
```

Obviously replace the values above with your own. And read [pghoard docs](https://github.com/ohmu/pghoard#configuration-keys) for more config explanation.

**Note:** Make sure you have enough space in your `/data`; use a Google Persistent Volume if you DB is very big.

Launch script which does 2 things:

1. Replaces our ENV variables with the right username and password for our replication (make sure you have enough connections for your replica user)
2. Launches the pghoard daemon.

```
#!/usr/bin/env bash

set -e

if [ -n "$TESTING" ]; then
    echo "Not running backup when testing"
    exit 0
fi

cat /pghoard.json.template | sed "s/\"password\": \"replica\"/\"password\": \"${REPLICA_PASSWORD}\"/" | sed "s/\"user\": \"replica\"/\"password\": \"${REPLICA_USER}\"/" > /pghoard.json
pghoard --config /pghoard.json
```

Once you build and upload your image to gcr.io you’ll need a replication controller to start your pghoard daemon pod:

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: pghoard
spec:
  replicas: 1
  selector:
    app: pghoard
  template:
    metadata:
      labels:
        app: pghoard
    spec:
        containers:
        - name: pghoard
          env:
            - name: REPLICA_USER
              value: "replicant"
            - name: REPLICA_PASSWORD
              value: "The tortoise lays on its back, its belly baking in the hot sun, beating its legs trying to turn itself over. But it can't. Not with out your help. But you're not helping."
          image: gcr.io/your-project/pghoard:latest
```

The reason I use a replication controller is because I want the pod to restart if it fails, if a simple pod is used it will stay dead and you’ll not have backups.

Future to do:

* Monitoring (are you backups actually done? if not, do you receive a notification?)
* Stats collection.
* Encryption of backups locally and then uploaded to the cloud (this is supported by pghoard).

Hope it helps, stay safe and sleep well at night.

Again, repo with the above: https://github.com/xarg/pghoard-k8s