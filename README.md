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

## Create Persistent Volumes and Statefulset

```shell
apiVersion: v1
kind: PersistentVolume
metadata:
  name: elastic-pv
spec:
  storageClassName: manual
  capacity:
    storage: 2Gi #Size of the volume
  accessModes:
    - ReadWriteOnce #type of access
  hostPath:
    path: "/tmp" #host location
----
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch-data
  namespace: picnic
  labels:
    app: elasticsearch-data
    env: dev
spec:
  serviceName: elasticsearch-data
  replicas: 2 # scale when desired
  selector:
    matchLabels:
      app: elasticsearch-data
  template:
    metadata:
      labels:
        app: elasticsearch-data
    spec:
      initContainers:
      - name: init-sysctl
        image: busybox:1.27.2
        command:
        - sysctl
        - -w
        - vm.max_map_count=262144
        securityContext:
          privileged: true
      containers:
      - name: elasticsearch-data
        image: elasticsearch:6.8.1
        imagePullPolicy: Always
        env:
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: DISCOVERY_SERVICE
          value: elasticsearch-discovery
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: CLUSTER_NAME
          value: elasticsearch
        - name: NODE_DATA
          value: "true"
        - name: NODE_MASTER
          value: "false"
        - name: NODE_INGEST
          value: "false"
        - name: HTTP_ENABLE
          value: "false"
        - name: ES_JAVA_OPTS
          value: -Xms256m -Xmx256m
        - name: PROCESSORS
          valueFrom:
            resourceFieldRef:
              resource: limits.cpu
        resources:
          limits:
            cpu: 1
        ports:
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: elasticsearch-data-storage
          mountPath: /tmp
      volumes:
      - name: elastic-pv
        persistentVolumeClaim:
         claimName: elastic-pv
      nodeSelector:
        role: data-client
        #volumeClaimTemplates:
        #- metadata:
        #name: elasticsearch-data-storage
        # spec:
        #storageClassName: slow
        #accessModes: [ ReadWriteOnce ]
        # volumeMode: Block
        #resources:
        #requests:
        #  storage: 2Gi # small for dev / testing

```

## Deploy Master Pods on Master Nodes

```shell

apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch-master
  namespace: picnic
  labels:
    app: elasticsearch-master
    env: dev
spec:
  replicas: 2 # scale as desired (see NUMBER_OF_MASTERS below)
  selector:
    matchLabels:
      app: elasticsearch-master
  template:
    metadata:
      labels:
        app: elasticsearch-master
        env: dev
    spec:
      initContainers:
      - name: init-sysctl
        image: busybox:1.27.2
        command:
        - sysctl
        - -w
        - vm.max_map_count=262144
        securityContext:
          privileged: true
      containers:
      - name: elasticsearch-master
        image: txn2/k8s-es:v6.2.3
        imagePullPolicy: Always
        env:
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: DISCOVERY_SERVICE
          value: elasticsearch-discovery
        - name: CLUSTER_NAME
          value: elasticsearch
        - name: NUMBER_OF_MASTERS
          value: "2"
        - name: NODE_MASTER
          value: "true"
        - name: NODE_INGEST
          value: "false"
        - name: NODE_DATA
          value: "false"
        - name: HTTP_ENABLE
          value: "false"
        - name: ES_JAVA_OPTS
          value: -Xms256m -Xmx256m
        - name: PROCESSORS
          valueFrom:
            resourceFieldRef:
              resource: limits.cpu
        resources:
          limits:
            cpu: 1
        ports:
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: storage
          mountPath: /data
      nodeSelector:
          role: master
      volumes:
      - emptyDir:
          medium: ""
        name: "storage"
        
  ```
        
## Deploy Clients Pods

``` shell
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch-client
  namespace: picnic
  labels:
    app: elasticsearch-client
    env: dev
spec:
  replicas: 2 # scale as desired
  selector:
    matchLabels:
      app: elasticsearch-client
  template:
    metadata:
      labels:
        app: elasticsearch-client
        env: dev
    spec:
      initContainers:
      - name: init-sysctl
        image: busybox:1.27.2
        command:
        - sysctl
        - -w
        - vm.max_map_count=262144
        securityContext:
          privileged: true
      containers:
      - name: elasticsearch-client
        image: elasticsearch:6.8.1
        imagePullPolicy: Always
        env:
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: DISCOVERY_SERVICE
          value: elasticsearch-discovery
        - name: CLUSTER_NAME
          value: elasticsearch
        - name: NODE_MASTER
          value: "false"
        - name: NETWORK_HOST
          value: "0.0.0.0"
        - name: NODE_INGEST
          value: "true"
        - name: NODE_DATA
          value: "false"
        - name: HTTP_ENABLE
          value: "true"
        - name: ES_JAVA_OPTS
          value: -Xms256m -Xmx256m
        - name: PROCESSORS
          valueFrom:
            resourceFieldRef:
              resource: limits.cpu
        resources:
          limits:
            cpu: 0.25
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        volumeMounts:
        - name: storage
          mountPath: /data
      nodeSelector:
        role: data-client
      volumes:
      - emptyDir:
          medium: ""
        name: "storage"
```

