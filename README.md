# stateful-mysql-kubernetes
Deploying a replicated MySQL cluster on Kubernetes with StatefulSets 

## Intro

In this project we are going to deploy a stateful MySQL cluster on Kubernetes.
We're going to assume we already have a Kubernetes cluster deployed in a cloud provider like AWS so that the focus of this project will be on configuring and deploying the StatefulSet.
A StatefulSet is a Kubernetes resource designed to manage stateful applications, providing stable and unique network identifiers, persistent storage, and ordered deployment and scaling for pods.
This is crucial for stateful applications like databases that require consistent network identity.

This project uses a single primary and two read replicas with an asynchronous replication scheme for MySQL. All database writes are handled by the primary. The database replicas receive data from the primary asynchronously. This means the primary will not wait for the data to be copied onto the replicas. This can improve the performance of the primary at the expense of having replicas that are not always exact copies of the primary.

## Declare the ConfigMap

We begin by declaring a ConfigMap to differentiate primary and replica MySQL pods. A ConfigMap in Kubernetes is an API object used to store non-sensitive configuration data in key-value pairs. It allows us to decouple configuration artifacts from container images, making the applications more portable and easier to configure.

The _master.cnf_ key enables binary logging, which allows the master node to record all database changes that will be replicated to slave nodes. The _slave.cnf_ key enforces read-only behavior.

Let's create the ConfigMap resource:

```
 kubectl create -f mysql-configmap.yaml
```

This ConfigMap will be referenced later in the StatefulSet declaration.


## Declare the Services

A Service in Kubernetes is an abstraction that defines a logical set of Pods and a policy by which to access them. It provides a stable network endpoint for accessing a set of Pods, solving several key networking challenges in a dynamic container environment and ensuring pods can always be addressed by a DNS name. Services can be of different types among which ClusterIP, which exposes the service on an internal IP within the cluster, and LoadBalancer, which creates an external load balancer (typically in cloud environments).

For the MySQL primary pod we are going to define a headless service,  a special type of Service that doesn't use a cluster IP address. Instead, it returns the IP addresses of individual Pods directly, which is particularly useful for stateful applications that require direct Pod communication.

Two services are defined:

- A headless service for pod DNS resolution. Because the service is named _mysql_, pods are accessible via _pod-name.mysql_.
- A service name _mysql-read_ to connect to for database reads. This service uses the default ServiceType of ClusterIP which assigns an internal IP address that load balances request to all the pods labeled with _app: mysql_.

To create the MySQL Services:

```
  kubectl create -f mysql-services.yaml
```


## Declare the Storage Class

We then declare a default storage class that will be used to dynamically provision general-purpose (gp2) EBS volumes for the Kubernetes Persistent Volumes. The built-in aws-ebs storage is specified along with the type gp2.
Let's create the storage class:

```
  kubectl create -f mysql-storageclass.yaml
```


## Declare the StatefulSet

Now it's time to piece together the StatefulSet.
There is a lot to unpack in the _mysql-statefulset.yaml_ file. Let's skip the bash scripts and focus on the highlights.

- `init-containers`: it starts up and runs to completion before any other container.
  - `init-mysql`: Assigns a unique MySQL server ID starting from 100 for the first pod and increments by one, as well as copying the appropriate configuration file from the config-map. The ID and appropriate configuration file are persisted on the conf volume.
  - `clone-mysql`: For pods after the primary, clone the database files from the preceding pod. The _xtrabackup_ tool performs the file cloning and persists the data on the data volume.
- `spec.containers`: declares the two main containers in the pod:
  - `mysql`: Runs the MySQL daemon and mounts the configuration in the conf volume and the data in the data volume.
  - `xtrabackup`: A sidecar container that provides additional functionality to the mysql container. It enables data cloning and begins replication on replicas using the cloned data files.
- `spec.volumes`:
  - `conf`: an empty directory that will be used for dynamic configuration files.
  - `config-map`:  a volume that references the ConfigMap _mysql_.
 
conf and config-map volumes are stored on the node's local disk. They are easily re-generated if a failure occurs and don't require Persistent Volumes.

- `volumeClaimTemplates`: A template for each pod that is used to create a Persistent Volume Claim. _ReadWriteOnceaccessMode_ allows the PV to be mounted by only one node at a time in read/write mode. The _storageClassName_ references the AWS EBS gp2 storage class named general that we created earlier. 
