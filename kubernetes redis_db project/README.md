# **Kubernetes Stateful Application: Persistent Redis**

This project provides a comprehensive, hands on demonstration of deploying and managing a stateful application (Redis) on Kubernetes. It showcases the complete lifecycle of a stateful service, from requesting persistent storage to verifying data survival across Pod restarts. This is a critical pattern for running any application that needs to save data, such as a database, cache, or message queue.

## **Table of Contents**

* [Core Concepts Explained](#core-concepts-explained)  
  * [Stateless vs Stateful Applications](#stateless-vs-stateful-applications)  
  * [The Kubernetes Persistent Storage Model](#the-kubernetes-persistent-storage-model)  
* [Repository Contents](#repository-contents)  
* [Prerequisites](#prerequisites)  
* [Step by Step Usage Instructions](#step-by-step-usage-instructions)  
* [Verification: The Proof of Persistence](#verification-the-proof-of-persistence)  
* [Cleanup](#cleanup)  
* [Conclusion](#conclusion)

## **Core Concepts Explained**

### **Stateless vs Stateful Applications**

A **stateless** application, like an NGINX web server, does not store any critical data. If a stateless pod is deleted, a new, identical one can be created without any data loss. A **stateful** application, like a database, must remember its data (its "state") between restarts. If a database pod is deleted, its data must be preserved. This project focuses on the proper management of stateful applications.

### **The Kubernetes Persistent Storage Model**

Kubernetes uses a powerful two part abstraction for managing storage, which decouples the application's *need* for storage from the underlying storage *implementation*.

1. **PersistentVolumeClaim (PVC)**: This is a **request for storage** made by an application. It's like filling out a request form specifying your needs: "I require at least 256Mi of storage that can be written to by a single pod at a time."  
2. **PersistentVolume (PV)**: This is a piece of actual storage in the cluster, like an AWS EBS volume or a local disk. It's like a pre existing, available "storage locker."

When a Pod is created with a PVC, Kubernetes finds a suitable PV and **binds** them together. The Pod then uses the PVC to access the storage. This binding remains even if the Pod is deleted, ensuring the new Pod can reconnect to the same storage and find its old data.

## **Repository Contents**

* **pvc.yaml**: Creates the PersistentVolumeClaim, our application's formal request for 256Mi of storage.  
* **deployment.yaml**: Deploys the Redis database application. This manifest contains the crucial volumes and volumeMounts sections that connect the Redis container to the PVC.  
* **service.yaml**: Creates a ClusterIP Service, providing a stable internal DNS name (redis-service) for other applications in the cluster to connect to the database.  
* **tester-pod.yaml**: Deploys a simple Ubuntu container that we use as a "testing station" to interact with our Redis service.

## **Prerequisites**

* A running Kubernetes cluster (e.g., minikube) with a storage provisioner enabled (minikube addons enable storage-provisioner).  
* The kubectl command line tool.  
* A container runtime like Docker.

## **Step by Step Usage Instructions**

1. Apply the Manifests  
   From your terminal, apply all four manifest files in the logical order of dependency.  
   \# 1\. Create the storage request  
   kubectl apply \-f pvc.yaml

   \# 2\. Deploy the database that uses the storage  
   kubectl apply \-f deployment.yaml

   \# 3\. Create the network endpoint for the database  
   kubectl apply \-f service.yaml

   \# 4\. Deploy our testing client  
   kubectl apply \-f tester-pod.yaml

   You can check that all pods are Running with kubectl get pods.

## **Verification: The Proof of Persistence**

This is the most critical part of the project. We will connect to the database, save data, destroy the database pod, and verify that the data is still there.

1. Get a Shell Inside the Tester Pod  
   Use kubectl exec to open an interactive bash shell in our testing container.  
   kubectl exec \-it redis-tester-pod \-- /bin/bash

2. Install Redis Tools and Connect  
   From inside the tester pod's shell, install the necessary client tools and connect to the Redis database using its stable service name.  
   \# Install redis-tools (one time setup)  
   apt-get update && apt-get install \-y redis-tools

   \# Connect to the Redis service  
   redis-cli \-h redis-service

   Your command prompt will change to redis-service:6379\>.  
3. Write and Read Data  
   Set a key value pair and then retrieve it to confirm the connection works.  
   SET mykey "Hello Persistent Redis"  
   GET mykey

   The server should reply with "Hello Persistent Redis". Type exit to leave the redis-cli.  
4. Destroy the Database Pod  
   Open a new, separate WSL terminal. Leave the one with the tester pod shell open. In the new terminal, find the name of your Redis pod and delete it.  
   \# Get the pod name  
   kubectl get pods \-l app=redis

   \# Delete the pod (replace with your pod's actual name)  
   kubectl delete pod \<redis-deployment-pod-name\>

   The Kubernetes Deployment will instantly see this and create a new, replacement Redis pod.  
5. The Final Proof  
   Go back to your first terminal (the one inside the tester pod). Reconnect to the database.  
   redis-cli \-h redis-service

   Now, try to retrieve the key you set earlier.  
   GET mykey

   The server will reply with "Hello Persistent Redis". This proves that even though the application pod was completely destroyed, the data was safely stored on the Persistent Volume and was re attached to the new pod, demonstrating true persistence.

## **Cleanup**

To remove all the resources created by this project, run the following commands:

kubectl delete deployment redis-deployment  
kubectl delete service redis-service  
kubectl delete pvc redis-pv-claim  
kubectl delete pod redis-tester-pod

## **Conclusion**

This project demonstrates a critical skill for any infrastructure engineer: managing stateful applications. By mastering the Persistent Volume and Persistent Volume Claim subsystem, you can reliably run databases and other stateful services on Kubernetes, ensuring data integrity and high availability.