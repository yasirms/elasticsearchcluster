# Elastic Search Cluster on Google Kubernetes Engine

This project is to demonstrate how to build a Elasticsearch Cluster on Kubernetes.

## Cluster Design
The Google Cloud Kubernetes engine has been used to create Kubernetes Cluster. The Cluster is managed by Google cloud. There are 4 nodes that have been provisioned as part of cluster configuration. This provisioning has to be done manually, though Terraform or some other DevOps tool can be used to provision the kubernetes clusters.

2 Nodes will work as Elasticsearch Master
2 Nodes will work as Data and Client nodes

2 nodes will be labeled as master and 2 nodes will be labeled data-client. This will help in scheduling master, data and client pods on the desired nodes.

Use below commands to label 2 nodes as master nodes:

```shell
kubectl label nodes <nodeid> role=master

```
Use below commands to label 2 nodes as data+client nodes:

```shell
kubectl label nodes <nodeid> role=data-client

```

The above labels have been used in the configuration below to select a node where to run the pod using <b> nodeSelector </b> option.

## Design Overview


# Cluster Configuration
## Creating new namespace

file: 01-namespace.yml

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

Below configuration creates 3 services. 

1. One service(elasticsearch-discovery) for Elasticsearch Master endpoint
2. One service(elasticsearch-data) for Elasticsearch for Elastic Data pods discovery
3. Once Service(elasticsearch-client) for Elasticsearch for Elastic Client pods discovery

All services expose necassary ports for external communication.

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

## Create Persistent Volume and Statefulset

File: 03-deploy-data.yml

Below configuration does following:
### 1 - Create Peristent Volume
It first creates Persistent Volume. Since data in pods are generally stateless by nature, to store data for session, we need to facilitate pods to be able to store data. The persistent is 2GB and is mounted on the "/tmp" on the host running the pod.

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
```

### 2. Create a Statefull Set
StatefulSet is the workload API object used to manage stateful applications. Manages the deployment and scaling of a set of Pods , and provides guarantees about the ordering and uniqueness of these Pods. Unlike a Deployment, a StatefulSet maintains a sticky identity for each of their Pods. These pods are created from the same spec, but are not interchangeable: each has a persistent identifier that it maintains across any rescheduling. It will manages states of sessions within Elastic Cluster.

The container is initialized with buzybox container image. Then Elasticsearch v6.8.1 is deployed. All the required environment variables for Elastic Data pods initialization are set. Necassary ports are exposed. VolumeClaim bind persistent storage(that was created in the last step) with the pod. 

Finally Node is selected where these data pods are going to run.


```shell
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

```

## Deploy Master Pods on Master Nodes

file: 04-deploy-master.yml

The following configuration creates a deployment on the master nodes. There are 2 repicas initially being created. The containers are initialized with the buzybox image. Then elasticsearch image v6.8.1 is pulled from dockerhub. The environment variables are set to initize the pods with correct configuraitons. Necassary ports are exposed. 

The pods are set to scheduled to run on nodes with lable "master".


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

file: 05-deploy-client.yml

The following configuration creates a deployment for handling client requests and exposing Elastic cluster to the clients. There are 2 repicas initially being created. The containers are initialized with the buzybox image. Then elasticsearch image v6.8.1 is pulled from dockerhub. The environment variables are set to initize the pods with correct configuraitons. Necassary ports are exposed. 

The pods are set to scheduled to run on nodes with lable "data-client".


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

