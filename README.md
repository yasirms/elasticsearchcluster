# Elastic Search Cluster on Google Kubernetes Engine

This project is to demonstrate how to build a Elasticsearch Cluster on Kubernetes.

## Cluster Design
The Google Cloud Kubernetes engine has been used to create Kubernetes Cluster. The Cluster is managed by Google cloud. There are 4 nodes that have been provisioned as part of cluster configuration.

2 Nodes will work as Elasticsearch Master
2 Nodes will work as Data and Client nodes

## Design Overview


# Cluster Configuration
## Creating new namespace

Below configuration creates new namespace for Elasticsearch Cluster.

```shell
#filename: 01-namespace.yml

apiVersion: v1
kind: Namespace
metadata:
        name: picnic
        labels:
                env: dev
```

##  Creating new Services

file: 02-service.yml

```shell
---
apiVersion: v1
kind: Service
metadata:
  namespace: picnic
  name: elasticsearch
  labels:
    env: dev
spec:
  type: ClusterIP
  selector:
    app: elasticsearch-client
  ports:
  - name: http
    port: 9200
    protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  namespace: picnic
  name: elasticsearch-data
  labels:
    env: dev
spec:
  clusterIP: None
  selector:
    app: elasticsearch-data
  ports:
  - port: 9300
    name: transport
 ---
apiVersion: v1
kind: Service
metadata:
  namespace: picnic
  name: elasticsearch-discovery
  labels:
    env: dev
spec:
  selector:
    app: elasticsearch-master
  ports:
  - name: transport
    port: 9300
    protocol: TCP

```
