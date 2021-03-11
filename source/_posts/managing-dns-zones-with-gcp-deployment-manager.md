---
title: Managing DNS zones with GCP deployment manager
date: 2018-08-21 14:54:02
categories:
tags: [gcp, dns]
hidden: true
---

I looked for an example of how to setup a managed DNS zone on GCP (using their deployment manager) with a Global IP address to no avail.
After some inspiration from this issue: https://github.com/GoogleCloudPlatform/deploymentmanager-samples/issues/62
I managed to create the following config:

 
### config.yaml

```
imports:
- path: templates/ip.py
- path: templates/dns.py

resources:
- name: global-ip
  type: templates/ip.py
  properties:
    name: global-ip 
    description: Global IP address used below in the DNS 

- name: example-com
  type: templates/dns.py
  properties:
    description: Example domain
    dnsName: example.com.
    resourceRecordSets:
    - name: "*.gorgias.io."
      type: TXT
      ttl: 3600
      rrdatas:
      - '"v=spf1 include:spf.gorgias.io -all"'
    - name: "*.gorgias.io."
      type: MX
      ttl: 3600
      rrdatas:
      - "10 mx1.gorgias.io."
      - "10 mx2.gorgias.io."
    - name: "*.gorgias.io."
      type: A
      ttl: 3600
      rrdatas:
      - "$(ref.global-ip.address)"

```

### templates/ip.py

```python
def GenerateConfig(context):
    return {'resources': [{
        'type': 'compute.v1.globalAddress',
        'name': context.env['name'],
        'properties': {
            'description': context.properties['description'],
        }
    }]}
```

### templates/dns.py

```python
def GenerateConfig(context):
    resources = [{
        'type': 'dns.v1.managedZone',
        'name': context.env['name'],
        'properties': {
            'description': context.properties['description'],
            'dnsName': context.properties['dnsName'],
        }
    }]

    for i, record in enumerate(context.properties['resourceRecordSets']):
        # Used to create the records
        resources.append({
            'name': 'dns-{}-create'.format(i),
            'action': 'gcp-types/dns-v1:dns.changes.create',
            'metadata': {
                'runtimePolicy': [
                    'CREATE',
                ],
            },
            'properties': {
                'managedZone': '$(ref.{}.name)'.format(context.env['name']),
                'additions': [
                    record,
                ],
            },
        })
        # Used on deployment teardown to delete the records 
        resources.append({
            'name': 'dns-{}-delete'.format(i),
            'action': 'gcp-types/dns-v1:dns.changes.create',
            'metadata': {
                'runtimePolicy': [
                    'DELETE',
                ],
            },
            'properties': {
                'managedZone': '$(ref.{}.name)'.format(context.env['name']),
                'deletions': [
                    record,
                ],
            },
        })

    return {'resources': resources}
```


