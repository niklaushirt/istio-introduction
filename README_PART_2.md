


# Introduction to Istio - Part B
## Traffic Management for your Microservices



# Prerequisites
See [Part A](./README.md) for setup instructions. 

# Steps


### Part B: Modify sample application 

Modify sample application to use an external datasource, deploy the application and Istio envoys with egress traffic enabled

5. [Create an external datasource for the application](#5-create-an-external-datasource-for-the-application)
6. [Modify sample application to use the external database](#6-modify-sample-application-to-use-the-external-database)
7. [Deploy application microservices and Istio envoys with egress traffic enabled](#7-deploy-application-microservices-and-istio-envoys-with-egress-traffic-enabled)


## Part B:  Modify sample application to use an external datasource, deploy the application and Istio envoys with egress traffic enabled

In this part, we will modify the sample BookInfo application to use use an external database, and enable egress traffic. Please ensure you have the Istio control plane installed on your Kubernetes cluster as mentioned in the prerequisites.

## 5. Create an external datasource for the application

Provision Compose for MySQL in IBM Cloud via https://console.ng.bluemix.net/catalog/services/compose-for-mysql  
Go to Service credentials and view your credentials. Your MySQL hostname, port, user, and password are under your credential uri and it should look like this
![images](images/mysqlservice.png)

## 6. Modify sample application to use the external database

In this step, the original sample BookInfo Application is modified to leverage a MySQL database. The modified microservices are the `details`, `ratings`, and `reviews`. This is done to show how Istio can be configured to enable egress traffic for applications leveraging external services outside the Istio data plane, in this case a database.

In this step, you can either choose to build your Docker images for different microservices from source in the [microservices folder](/microservices) or use the given images.
> For building your own images, go to [microservices folder](/microservices)

The following modifications were made to the original Bookinfo application. The **details microservice** is using Ruby and a `mysql` ruby gem was added to connect to a MySQL database. The **ratings microservice** is using Node.js and a `mysql` module was added to connect to a MySQL database. The **reviews v1,v2,v3 microservices** is using Java and a `mysql-connector-java` dependency was added in [build.gradle](/microservices/reviews/reviews-application/build.gradle) to connect to a MySQL database. The `reviews`
service runs inside an OpenLiberty container running on OpenJ9. More source code was added to [details.rb](/microservices/details/details.rb), [ratings.js](/microservices/ratings/ratings.js), [LibertyRestEndpoint.java](/microservices/reviews/reviews-application/src/main/java/application/rest/LibertyRestEndpoint.java) that enables the application to use the details, ratings, and reviews data from the MySQL Database.  

Preview of added source code for `ratings.js` for connecting to MySQL database:
![ratings_diff](images/ratings_diff.png)


You will need to update the `secrets.yaml` file to include the credentials provided by IBM Cloud Compose.

> Note: The values provided in the secrets file should be run through `base64` first.

```bash
echo -n <username> | base64
echo -n <password> | base64
echo -n <host> | base64
echo -n <port> | base64
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: demo-credentials
type: Opaque
data:
  username: YWRtaW4=
  password: VEhYTktMUFFTWE9BQ1JPRA==
  host: c2wtdXMtc291dGgtMS1wb3J0YWwuMy5kYmxheWVyLmNvbQ==
  port: MTg0ODE=
```

Once the secrets are set add them to your Kubernetes cluster:

```bash
$ kubectl apply -f secrets.yaml
```

You can verify the values of the keys in the secrets object for the mysql database with:

```bash
$ kubectl get secret demo-credentials -o json | grep -A4 '"data"'
    "data": {
        "host": "c2wtdXMtc291dGgtMS1wb3J0YWwuMzguZGJsYXllci5jb20=",
        "password": "T0NVUUhDQ1NKT0JEVEtUWQ==",
        "port": "NTk0NTQ=",
        "username": "YWRtaW4="
```

## 7. Deploy application microservices and Istio envoys with Egress traffic enabled

By default, Istio-enabled applications will be unable to access URLs outside of the cluster. All outbound traffic in the pod are redirected by its sidecar proxy which only handles destinations inside the cluster.

Istio allows you to define a `ServiceEntry` to control egress to external services.  We've defined a simple egress configuration using a `ServiceEntry` to allow services to talk to the MySQL Compose instance.  In the `MySQL-egress.yaml` file, change the `host` and `number` fields to the hostname and port provided in the Compose connection string, and then use `kubectl` to apply the changes.


```
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: MySQL-cloud
spec:
  hosts:
  - sl-us-south-1-portal.38.dblayer.com
  ports:
  - number: 59454
    protocol: tcp
  location: MESH_EXTERNAL
```

```bash
$ kubectl apply -f mysql-egress.yaml

```

* Insert data in your MySQL database in IBM Cloud.
> This inserts the database design and initial data for the database.

```bash
$ kubectl apply -f mysql-data.yaml
```

As an initial step, remove the ingress rules from the sample app, as they are
not compatible with the MySQL demo portion:

```
kubectl delete -f ~/istio/samples/bookinfo/networking/bookinfo-gateway.yaml
```

* Deploy `productpage` with Envoy injection and the `gateway` for products and reviews.  

```bash
$ kubectl apply -f <(istioctl kube-inject -f bookinfo.yaml)
$ kubectl apply -f bookinfo-gateway.yaml
```

* Deploy `details` with Envoy injection and Egress traffic enabled.  

```bash
$ kubectl apply -f <(istioctl kube-inject -f details-new.yaml)
```

Note that Kubernetes `apply` shuts down the old pod and replaces it with the new one.

```bash
$ kubectl get pods
NAME                              READY     STATUS            RESTARTS   AGE
details-v1-76df85799c-njdk7       2/2       Terminating       0          5d
details-v1-86f56ff4d8-cc9fr       0/2       PodInitializing   0          7s
productpage-v1-5c67c7d4d7-4mkjm   2/2       Running           0          2m
ratings-v1-648467b449-4f7kp       2/2       Running           0          5d
reviews-v1-76ff8854fc-n5b6l       2/2       Running           0          5d
reviews-v2-65cb86568c-nqhqk       2/2       Running           0          5d
reviews-v3-995b68dcc-j67hf        2/2       Running           0          5d
setup                             0/1       Completed         0          3m
```

* Deploy `reviews` with Envoy injection and Egress traffic enabled.  

```bash
$ kubectl apply -f <(istioctl kube-inject -f reviews-new.yaml)
```

* Deploy `ratings` with Envoy injection and Egress traffic enabled.  

```bash
$ kubectl apply -f <(istioctl kube-inject -f ratings-new.yaml)
```

You can now access your application to confirm that it is getting data from your MySQL database.
Point your browser to:  
`http://${GATEWAY_URL}/productpage`

# Clean-up


* To delete the BookInfo app and its route-rules: ` ~/istio/samples/bookinfo/platform/kube/cleanup.sh`

* To delete Istio from your cluster

```bash
kubectl delete -f https://raw.githubusercontent.com/niklaushirt/microservices-traffic-management-using-istio/master/istio.yaml
kubectl delete -f ~/istio/install/kubernetes/helm/istio/templates/crds.yaml
kubectl delete ns istio-system

```

# References
[Istio.io](https://istio.io/docs/tasks/)
# License
This code pattern is licensed under the Apache Software License, Version 2.  Separate third party code objects invoked within this code pattern are licensed by their respective providers pursuant to their own separate licenses. Contributions are subject to the [Developer Certificate of Origin, Version 1.1 (DCO)](https://developercertificate.org/) and the [Apache Software License, Version 2](http://www.apache.org/licenses/LICENSE-2.0.txt).

[Apache Software License (ASL) FAQ](http://www.apache.org/foundation/license-faq.html#WhatDoesItMEAN)
