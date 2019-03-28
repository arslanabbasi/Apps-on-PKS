# Running Contour as Ingress on PKS k8s clusters with NSX-T

## Pre-requisites

Before performing the procedures in this topic, you must have installed and configured the following:

- PKS v1.2+
- NSX-T v2.3+
- A PKS plan with at least 1 master and 2 worker nodes
- Make sure that the k8s cluster is deployed with priviliged access. Deployment of nginx will fail otherwise

## Overview

Contour is an Ingress controller for Kubernetes that works by deploying the Envoy proxy as a reverse proxy and load balancer. Unlike other Ingress controllers, Contour supports dynamic configuration updates out of the box while maintaining a lightweight profile.

## Install Contour

Follow the steps below to run Contour on k8s, side by side NSX-T. Contour will be exposed outside using virtual servers on NSX-T but Contour will be performing the ingress functionality. NSX-T will be acting as a L4 LB and will just be forwarding all the traffic to Contour.


### Step 1: Create the Contour Deployment and the Service

In this tutorial, we will deploy Contour as a deployment. For more complex deployment configuration, please refer the Contour github [deployment-options](https://github.com/heptio/contour/blob/master/docs/deploy-options.md) page.

We will use the command below to deploy Contour including the Namespace, ServiceAccount, RBAC rules, Cnotour deployment and service. Run the following command from within the contour-ingress directory.

```
$ kubectl apply -f deployment/.
```

There are 4 files in the deployment folder
- `01-common.yaml`: Creates the `heptio-contour` Namespace and a ServiceAccount.
- `02-rbac.yaml`: Creates the RBAC rules for Contour. The Contour RBAC permissions are the minimum required for Contour to operate.
- `02-contour.yaml`: Runs the Contour pods with either the DaemonSet or the Deployment. See Architecture for pod details.
- `02-service.yaml`: Creates the Service object so that Contour can be reached from outside the cluster.


### Step 2: Check the Contour PODs

```
$ kubectl get pods --all-namespaces -l app=contour
    NAMESPACE        NAME                       READY   STATUS    RESTARTS   AGE
    heptio-contour   contour-86d66889d9-ftpw6   2/2     Running   0          5d23h
    heptio-contour   contour-86d66889d9-kzljc   2/2     Running   0          5d23h
```
The status of Contour PODs is `Running` which means the Contour was deployed susccessfully


### Step 3: Retrieve Contour's IP

Run the following command to get Contour's IP

```
$ kubectl get -n heptio-contour service contour -o wide

    NAME      TYPE           CLUSTER-IP       EXTERNAL-IP               PORT(S)                      AGE     SELECTOR
    contour   LoadBalancer   10.100.200.153   100.64.64.5,172.32.0.50   80:31311/TCP,443:30308/TCP   5d23h   app=contour
```
Note down the external IP of the ingress-nginx for your environment. In this case, the Contour ingress can be reached at `172.32.0.50` and is listening on port 80 and 443.


### Step 4: Deploying the cafe application

1. Executing the following commands to deploy the cafe application
    ```
    $ kubectl apply -f complete-example/cafe.yaml
        deployment.extensions/coffee created
        service/coffee-svc created
        deployment.extensions/tea created
        service/tea-svc created

    $ kubectl apply -f complete-example/cafe-secret.yml
        secret/cafe-secret created
    ```
    
3. Deploy the ingress resource for cafe application. Make sure to change the host and hosts value in complete-example/cafe-ingress.yml file to reflect your environment
    ```
    $ kubectl apply -f complete-example/cafe-ingress.yml
        ingress.extensions/cafe-ingress created
    ```
    Note that there is a annotation in the file 'kubernetes.io/ingress.class: "contour"' which is essentially being listened by Contour on the api server. When Contour sees this annotation, it does the ingress for this service.

4. Check the POD status to verify that cafe application deployed successfully
    ```
    $ kubectl get pods

        NAME                                        READY     STATUS    RESTARTS   AGE
        coffee-56668d6f78-rzj27                     1/1       Running   0          2m46s
        coffee-56668d6f78-wxvvv                     1/1       Running   0          2m46s
        nginx-ingress-controller-56c5c48c4d-b4hsp   1/1       Running   0          18h
        tea-85f8bf86fd-bskzx                        1/1       Running   0          2m46s
        tea-85f8bf86fd-wcmqx                        1/1       Running   0          2m46s
        tea-85f8bf86fd-xw68j                        1/1       Running   0          2m46s
    ```
    All pods are showing `Running` which shows that cafe application was deployed successfully

### Step 5: Testing connectivity using the cafe application deployed in step 4

The following commands will test the connectivity externally to verify that our cafe application is reachable using the nginx ingress LB.

1. Populate the IC_IP and IC_HTTPS_PORT variable for the ingress controller. The ingress controller ip(IC_IP) was retrieved in step 5. The cafe application is using port 443 for https traffic

    ```
    $ IC_IP=172.32.0.50
    $ IC_HTTPS_PORT=443
    ```

2. Test the coffe PODs

    Issue the command below to curl your PODs. Note that there is coffee in the url which nginx controller is using to direct traffic to the coffee backend PODs. Issuing the command multiple time round robins the request to the 2 coffee backend PODs as defined in cafe.yaml. The "Server address" field in the curl output identifies the backend POD fullfilling the request

    ```
    $ curl --resolve cafe.lab.local:$IC_HTTPS_PORT:$IC_IP https://cafe.lab.local:$IC_HTTPS_PORT/coffee --insecure
    Server address: 172.25.3.8:80
    Server name: coffee-56668d6f78-wxvvv
    Date: 15/Mar/2019:19:05:54 +0000
    URI: /coffee
    Request ID: 242a10438ab9cc8c93b531db656e9b01
    
    $ curl --resolve cafe.lab.local:$IC_HTTPS_PORT:$IC_IP https://cafe.lab.local:$IC_HTTPS_PORT/coffee --insecure
    Server address: 172.25.3.9:80
    Server name: coffee-56668d6f78-rzj27
    Date: 15/Mar/2019:19:05:55 +0000
    URI: /coffee
    Request ID: 6d8bafb54e5c7a1c495e0790516cfa88
    ```
    

3. Test the tea PODs

    The cafe.yaml file deployed 3 replicas of the tea POD so issuing the curl command multiple time distributes the request on these 3 PODs. This can be verified using the "Server address" field in the outputs below.

    ```
    $ curl --resolve cafe.lab.local:$IC_HTTPS_PORT:$IC_IP https://cafe.lab.local:$IC_HTTPS_PORT/tea --insecure
    Server address: 172.25.3.10:80
    Server name: tea-85f8bf86fd-bskzx
    Date: 15/Mar/2019:19:12:23 +0000
    URI: /tea
    Request ID: e3ca80b2254fc47a96735b99615ebfb4
    
    $ curl --resolve cafe.lab.local:$IC_HTTPS_PORT:$IC_IP https://cafe.lab.local:$IC_HTTPS_PORT/tea --insecure
    Server address: 172.25.3.11:80
    Server name: tea-85f8bf86fd-wcmqx
    Date: 15/Mar/2019:19:12:24 +0000
    URI: /tea
    Request ID: 546e2deb5e6dc0f11cc677e21b764976
    
    $ curl --resolve cafe.lab.local:$IC_HTTPS_PORT:$IC_IP https://cafe.lab.local:$IC_HTTPS_PORT/tea --insecure
    Server address: 172.25.3.12:80
    Server name: tea-85f8bf86fd-xw68j
    Date: 15/Mar/2019:19:12:25 +0000
    URI: /tea
    Request ID: ef81fc9439705a2990d3984ec0a0464e
    ```

    Alternatively, a DNS entry can be added for cafe.lab.local(hostname used in my environment) to map to 172.26.80.100 to access the url directly from the browser.


References:
- https://github.com/heptio/contour