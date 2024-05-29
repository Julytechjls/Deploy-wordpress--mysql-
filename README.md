Deploying WordPress and MySQL with Persistent Volumes

This tutorial shows you how to deploy a WordPress site and a MySQL database using Minikube. Both applications use PersistentVolumes and PersistentVolumeClaims to store data.

A PersistentVolume (PV) is a piece of storage in the cluster that has been manually provisioned by an administrator, or dynamically provisioned by Kubernetes using a StorageClass. A PersistentVolumeClaim (PVC) is a request for storage by a user that can be fulfilled by a PV. PersistentVolumes and PersistentVolumeClaims are independent from Pod lifecycles and preserve data through restarting, rescheduling, and even deleting Pods.

Objectives

Create PersistentVolumeClaims and PersistentVolumes
Create a kustomization.yaml with
a Secret generator
MySQL resource configs
WordPress resource configs
Apply the kustomization directory by kubectl apply -k ./
Clean up

Before you begin

You need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. It is recommended to run this tutorial on a cluster with at least two nodes that are not acting as control plane hosts. If you do not already have a cluster, you can create one by using minikube or you can use one of these Kubernetes playgrounds:

Killercoda
Play with Kubernetes
To check the version, enter kubectl version.
The example shown on this page works with kubectl 1.27 and above.

Download the following configuration files:

mysql-deployment.yaml

wordpress-deployment.yaml

Creating a kustomization.yaml

Add a Secret generator
A Secret is an object that stores a piece of sensitive data like a password or key. Since 1.14, kubectl supports the management of Kubernetes objects using a kustomization file. You can create a Secret by generators in kustomization.yaml.

Add a Secret generator in kustomization.yaml from the following command. You will need to replace YOUR_PASSWORD with the password you want to use.

apiVersion: v1
kind: Secret
metadata:  
    name: test-secret
data:  
    username: bXktYXBw  
    password: Mzk1MjgkdmRnN0pi

After running the first one since it previously gave error, the you can now run this to overwrite:
secretGenerator:
- name: mysql-pass
  literals:
  - password="mysql1234"
resources:
  - mysql-deployment.yaml
  - wordpress-deployment.yaml

 Add resource configs for MySQL and WordPress
The following manifest describes a single-instance MySQL Deployment. The MySQL container mounts the PersistentVolume at /var/lib/mysql. The MYSQL_ROOT_PASSWORD environment variable sets the database password from the Secret.

apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:8.0
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        - name: MYSQL_DATABASE
          value: wordpress
        - name: MYSQL_USER
          value: wordpress
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim

he following manifest describes a single-instance WordPress Deployment. The WordPress container mounts the PersistentVolume at /var/www/html for website data files. The WORDPRESS_DB_HOST environment variable sets the name of the MySQL Service defined above, and WordPress will access the database by Service. The WORDPRESS_DB_PASSWORD environment variable sets the database password from the Secret kustomize generated.

apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
  type: LoadBalancer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:6.2.1-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        - name: WORDPRESS_DB_USER
          value: wordpress
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wp-pv-claim


Apply and Verify
The kustomization.yaml contains all the resources for deploying a WordPress site and a MySQL database. You can apply the directory by

kubectl apply -k ./
Now you can verify that all objects exist.

Verify that the Secret exists by running the following command:

kubectl get secrets
The response should be like this:

NAME                    TYPE                                  DATA   AGE
mysql-pass-c57bb4t7mf   Opaque                                1      9s
Verify that a PersistentVolume got dynamically provisioned.

kubectl get pvc
Note:
It can take up to a few minutes for the PVs to be provisioned and bound.
The response should be like this:

NAME             STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
mysql-pv-claim   Bound     pvc-8cbd7b2e-4044-11e9-b2bb-42010a800002   20Gi       RWO            standard           77s
wp-pv-claim      Bound     pvc-8cd0df54-4044-11e9-b2bb-42010a800002   20Gi       RWO            standard           77s
Verify that the Pod is running by running the following command:

kubectl get pods
Note:
It can take up to a few minutes for the Pod's Status to be RUNNING.
The response should be like this:

NAME                               READY     STATUS    RESTARTS   AGE
wordpress-mysql-1894417608-x5dzt   1/1       Running   0          40s
Verify that the Service is running by running the following command:

kubectl get services wordpress
The response should be like this:

NAME        TYPE            CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
wordpress   LoadBalancer    10.0.0.89    <pending>     80:32406/TCP   4m
Note:
Minikube can only expose Services through NodePort. The EXTERNAL-IP is always pending.
Run the following command to get the IP Address for the WordPress Service:

minikube service wordpress --url
The response should be like this:

http://1.2.3.4:32406
Copy the IP address, and load the page in your browser to view your site.

You should see the WordPress set up page similar to the following screenshot.

wordpress-init

![image](https://github.com/Julytechjls/Deploy-wordpress--mysql-/assets/166651033/1c25de47-59e4-4e89-a707-5f33a1faa219)


