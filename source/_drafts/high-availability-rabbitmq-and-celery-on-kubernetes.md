---
title: High Availability RabbitMQ and Celery on Kubernetes
date: 2017-12-24 16:52:03
categories: devops 
tags: [kubernetes, rabbitmq, celery, ha]
---

Kubernetes, RabbitMQ and Celery provides a very natural way to create a reliable python worker cluster. 

These are my notes on how to achieve a basic cluster. Here's the breakdown of our steps:

- Configure our docker containers for RabbitMQ
- Test locally RabbitMQ cluster using docker-compose.
- Configure Celery App to use multiple workers.
- Create the replication controllers and services for Kubernetes.

If you worked with celery you know that it requires a message broker that it uses to distribute tasks across machines. The recommended one is RabbitMQ which we are going to use.

Because we want to run our RabbitMQ cluster on Kubernetes first we'll have to make the Docker containers for it. Here's a sample Dockerfile that you can use:

RabbitMQ Dockerfile
--


Local cluster
--

Configure Celery to use multiple hosts
--

Kill stuff and see what happens
--

Maximum paranoia: ACTIVATED
--

One very important sad thing to remember is that you have to expect your workers will die at any moment and always code with that in mind. IMHO this is the hardest part of all.

At Gorgias we're sending tons of emails and also making HTTP requests to user defined http endpoints before sending the said emails which can fail with a timeout or an error. These are just a few questions that arise: 

- Should the emails be sent if other parts of the process have passed or not? 
- What happens if the server responded with a 200 code but actually didn't accepted the request?
- What if the worker is killed in the middle of the transaction with the mail server? Was the email sent or not? Should we retry?
- If the mail server is down do we have a retry mechanism and when should the retry switch to a different server?

Check out the whole example project.

