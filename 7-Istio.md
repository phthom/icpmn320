![icp000](images/icp000.png)

---

![](./images/istio.png)

# Istio Lab

In recent years, with the development of container technology, more enterprise customers are turning to microservices. Microservices are a combination of lightweight and fine-grained services that work cohesively to allow for larger, application-wide functionality. This approach improves modularity and makes applications easier to develop and test when compared to traditional, monolithic application. With the adoption of microservices, new challenges emerge due to a myriad of services that exist in larger systems. Developers must now account for service discovery, load balancing, fault tolerance, dynamic routing, and communication security. Thanks to Istio, we can turn disparate microservices into an integrated service mesh by systemically injecting envoy proxy into the network layers while decoupling the operators to connect, manage, and secure microservices for application feature development.

This lab takes you step-by-step through the installation of Istio and the deployment of microservices-based applications in IBM Cloud Private.

> **Prerequisites** : you should be logged on your VM and connected to your ICP master.


### Table of Contents

---
- [Task1: Installing Istio on IBM Cloud Private](#task1--installing-istio-on-ibm-cloud-private)
- [Task2 - Deploy the Bookinfo application](#task2---deploy-the-bookinfo-application)
    + [Create Secret](#create-secret)
    + [Prepare the Bookinfo manifest](#prepare-the-bookinfo-manifest)
    + [Automatic Sidecar Injection](#automatic-sidecar-injection)
- [Task3: Access the Bookinfo application](#task3--access-the-bookinfo-application)
- [Task4: Collect Metrics with Prometheus](#task4--collect-metrics-with-prometheus)
- [Task5: Visualizing Metrics with Grafana](#task5--visualizing-metrics-with-grafana)
- [Congratulations](#congratulations)

---



# Introduction

This example deploys a sample application composed of four separate microservices used to demonstrate various Istio features. The application displays information about a book, similar to a single catalog entry of an online book store. Displayed on the page is a description of the book, book details (ISBN, number of pages, and so on), and a few book reviews.

The Bookinfo application is broken into four separate microservices:

- `productpage`. The `productpage` microservice calls the `details` and `reviews` microservices to populate the page.
- `details`. The `details` microservice contains book information.
- `reviews`. The `reviews` microservice contains book reviews. It also calls the `ratings` microservice.
- `ratings`. The `ratings` microservice contains book ranking information that accompanies a book review.

There are 3 versions of the `reviews` microservice:

- Version v1 doesn’t call the `ratings` service.
- Version v2 calls the `ratings` service, and displays each rating as 1 to 5 black stars.
- Version v3 calls the `ratings` service, and displays each rating as 1 to 5 red stars.

To run the sample with Istio requires no changes to the application itself. Instead, we simply need to configure and run the services in an Istio-enabled environment, with Envoy sidecars injected along side each service. The needed commands and configuration vary depending on the runtime environment although in all cases the resulting deployment will look like this:

![image-20181202151511591](images/image-20181202151511591.png)





# Task 1: Installing Istio on IBM Cloud Private 

Istio has been normal installed during your IBM Cloud Private installation (parameter "istio: enabled" in the config.yaml). You can also install istio after the IBM Cloud Private installation. 

Be sure you are connected to your environment :

```bash
cloudctl login -a https://$CLUSTERNAME.icp:8443 --skip-ssl-validation -u admin -p $CLUSTERPASS -n default
helm version --tls
```

An `istio-system` namespace with the `ibm-privileged-psp` has been already created to support ISTIO.

`kubectl get namespaces`

Results:

```bash
# kubectl get namespaces
NAME                                STATUS   AGE
cert-manager                        Active   8h
default                             Active   8h
ibmcom                              Active   8h
icp-system                          Active   8h
istio-system                        Active   8h
kube-public                         Active   8h
kube-system                         Active   8h
microclimate                        Active   50m
microclimate-pipeline-deployments   Active   50m
multicluster-endpoint               Active   8h
services                            Active   7h58m
```

Add a new helm repo to the cluster: 

```helm repo add ibm-charts https://raw.githubusercontent.com/IBM/charts/master/repo/stable/```

Install Istio with the following helm command:

```bash
helm install ibm-charts/ibm-istio --name istio --namespace istio-system --tls --set prometheus.enabled=true --set grafana.enabled=true --set kiali.enabled=true --version=1.1.7  
```

Results:

```
... 
==> v1beta1/PodDisruptionBudget
NAME                  MIN AVAILABLE  MAX UNAVAILABLE  ALLOWED DISRUPTIONS  AGE
istio-galley          1              N/A              0                    2s
istio-ingressgateway  1              N/A              0                    2s
istio-telemetry       1              N/A              0                    2s
istio-pilot           1              N/A              0                    2s

==> v1/ClusterRoleBinding
NAME                                                    AGE
istio-galley-admin-role-binding-istio-system            1s
istio-ingressgateway-istio-system                       1s
istio-kiali-admin-role-binding-istio-system             1s
istio-mixer-admin-role-binding-istio-system             1s
istio-pilot-istio-system                                1s
prometheus-istio-system                                 1s
istio-citadel-istio-system                              1s
istio-sidecar-injector-admin-role-binding-istio-system  1s

==> v1/Pod(related)
NAME                                    READY  STATUS             RESTARTS  AGE
istio-galley-7f96856f47-m2wft           0/1    ContainerCreating  0         1s
istio-ingressgateway-76ffddbdc6-4wp8k   0/1    ContainerCreating  0         1s
grafana-56db589975-qjbvt                0/1    ContainerCreating  0         1s
kiali-6cbc64b89f-mlp9s                  0/1    ContainerCreating  0         1s
istio-telemetry-7bfd6c569b-m9r95        0/2    ContainerCreating  0         1s
istio-pilot-59f4fd44c-d25fm             0/2    ContainerCreating  0         1s
prometheus-ff8f649c9-tml8b              0/1    ContainerCreating  0         1s
istio-citadel-578dcb6fb4-rbdr6          0/1    Pending            0         1s
istio-sidecar-injector-b5b49647f-4hf6w  0/1    Pending            0         1s


NOTES:
Thank you for installing ibm-istio.

Your release is named istio.

To get started running application with Istio, execute the following steps:
1. Label namespace that application object will be deployed to by the following command (take default namespace as an example)

$ kubectl label namespace default istio-injection=enabled
$ kubectl get namespace -L istio-injection

2. Deploy your applications

$ kubectl apply -f <your-application>.yaml

For more information on running Istio, visit:
https://istio.io/
```



Be sure to be at the **version 1.1.7**. This command installs ISTIO to add Prometheus and grafana in the scope for this exercise.



Ensure that the `istio-*` Kubernetes services are deployed before you continue.

```bash
kubectl get svc -n istio-system
```
Output:

```console
# kubectl get svc -n istio-system
NAME                     TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                                                                                                                      AGE
grafana                  ClusterIP      10.0.206.68    <none>        3000/TCP                                                                                                                                     2m25s
istio-citadel            ClusterIP      10.0.192.173   <none>        8060/TCP,15014/TCP                                                                                                                           2m25s
istio-galley             ClusterIP      10.0.148.52    <none>        443/TCP,15014/TCP,9901/TCP                                                                                                                   2m25s
istio-ingressgateway     LoadBalancer   10.0.193.138   <pending>     15020:32468/TCP,80:31380/TCP,443:31390/TCP,31400:31400/TCP,15029:31058/TCP,15030:30207/TCP,15031:32694/TCP,15032:30116/TCP,15443:31463/TCP   2m25s
istio-pilot              ClusterIP      10.0.202.164   <none>        15010/TCP,15011/TCP,8080/TCP,15014/TCP                                                                                                       2m25s
istio-sidecar-injector   ClusterIP      10.0.83.73     <none>        443/TCP                                                                                                                                      2m25s
istio-telemetry          ClusterIP      10.0.52.231    <none>        9091/TCP,15004/TCP,15014/TCP,42422/TCP                                                                                                       2m25s
kiali                    ClusterIP      10.0.36.127    <none>        20001/TCP                                                                                                                                    2m25s
prometheus               ClusterIP      10.0.133.136   <none>        9090/TCP                                                                                                                                     2m25s
```
  **Note: the istio-ingressgateway service will be in `pending` state with no external ip. That is normal.**

Ensure the corresponding pods `istio-citadel-*`, `istio-ingressgateway-*`, `istio-pilot-*`, and `istio-policy-*` are all in **`Running`** state before you continue.

```bash
kubectl get pods -n istio-system
```
Output:

```console
# kubectl get pods -n istio-system
NAME                                     READY   STATUS    RESTARTS   AGE
grafana-56db589975-qjbvt                 1/1     Running   0          2m55s
istio-citadel-578dcb6fb4-rbdr6           1/1     Running   0          2m55s
istio-galley-7f96856f47-m2wft            1/1     Running   0          2m55s
istio-ingressgateway-76ffddbdc6-4wp8k    1/1     Running   1          2m55s
istio-pilot-59f4fd44c-d25fm              2/2     Running   1          2m55s
istio-sidecar-injector-b5b49647f-4hf6w   1/1     Running   0          2m55s
istio-telemetry-7bfd6c569b-m9r95         2/2     Running   4          2m55s
kiali-6cbc64b89f-mlp9s                   1/1     Running   0          2m55s
prometheus-ff8f649c9-tml8b               1/1     Running   0          2m55s
```

Before your continue, make sure all the pods are deployed and **`Running`**. If they're in `pending` state, wait a few minutes to let the deployment finish.

**Congratulations**! You successfully installed Istio into your cluster.



> **Important** **Note**: If you get some **problems** with Istio (not using the good version for example or some images are not being pulled ) then you need to delete all istio components. Then re-install Istio from the catalog and loop to task 1. 

```
helm delete istio --purge --tls 
```




# Task 2 - Deploy the Bookinfo application
If the **control plane** is deployed successfully, you can then start to deploy your applications that are managed by Istio. I will use the **Bookinfo** application as an example to illustrate the steps of deploying applications that are managed by Istio.

### Create a Secret

If you are using a private registry for the sidecar image, then you need to create a Secret of type docker-registry in the cluster that holds authorization token, and patch it to your application’s ServiceAccount. Use the following 2 commands (replace mycluster.icp with yours):

```bash 
kubectl create secret docker-registry private-registry-key --docker-server=$CLUSTERPASS:8500 --docker-username=admin --docker-password=$CLUSTERPASS --docker-email=null
```

For example:

```bash
kubectl -n default create secret docker-registry private-registry-key --docker-server=nicemmm.icp:8500 --docker-username=admin --docker-password=admin1! --docker-email=null
```

Then patch the service account:

```bash
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "private-registry-key"}]}'
```



### Prepare the Bookinfo manifest

Return to your directory:

`cd`

Create a new YAML file named **bookinfo.yaml** to save the Bookinfo application manifest.

This file is located in the **instructor's GitHub**.

> **Don't create the kurbenetes resources yet**.



### Automatic Sidecar Injection

If you have enabled automatic sidecar injection, the istio-sidecar-injector automatically injects Envoy **proxy** containers into your **application pods** that are running in the namespaces, labelled with istio-injection=enabled. 

Let's deploy the Bookinfo application to the default namesapce.

```bash
kubectl label namespace default istio-injection=enabled
```

Then add an image policy. To add a <u>Cluster Image Policy</u>, go to the **Menu > Manage > Resource Security**

![image-20181128220823372](images/image-20181128220823372.png)



Now let's add a new policy for our LDAP image and click on the **Create Image Policy**:

Fill the name field with **istio-policy** 

Then click **add a registry policy** and type : `docker.io/istio*` and `docker.io/istio/*`

Finish by clicking **Add** at the top right.

![image-20190709001545987](images/image-20190709001545987-2624146.png)

Now, here is the command to inject the sidecar when deploying the application:

`kubectl create -n default -f bookinfo.yaml`

Results:

```console
kubectl create -n default -f bookinfo.yaml
service "details" created
deployment.extensions "details-v1" created
service "ratings" created
deployment.extensions "ratings-v1" created
service "reviews" created
deployment.extensions "reviews-v1" created
deployment.extensions "reviews-v2" created
deployment.extensions "reviews-v3" created
service "productpage" created
deployment.extensions "productpage-v1" created
ingress.extensions "gateway" created
```

To check that the injection was successful, go to ICP console, click on the **Menu>Workload>Deployments**

![image-20181202144556442](images/image-20181202144556442.png)

Go to **productpage** deployment, then **drill down to POD** and then **drill down to containers.** You should have 2 containers in one POD. The istio-proxy is the side-car container running aside the application container. 

![image-20190709001747300](images/image-20190709001747300-2624267.png)



Now that the Bookinfo services are up and running, you need to make the application accessible from outside of your Kubernetes cluster, e.g., from a browser. An [Istio Gateway](https://istio.io/docs/concepts/traffic-management/#gateways) is used for this purpose.

**Cut and paste** the following command :

```bash
kubectl create -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
---
EOF
```

Check the gateway is running:

```bash
kubectl get gateway
```

Results:

```console
# kubectl get gateway
NAME               AGE
bookinfo-gateway   22s
```





# Task3: Access the Bookinfo application

After all pods for the Bookinfo application are in a running state and the gateway has been created,  you can access the Bookinfo **product page**. 

Display the service:

``` bash
kubectl get svc istio-ingressgateway -n istio-system
```

Results:

```console
 # kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)                                                                                                                   AGE
istio-ingressgateway   LoadBalancer   10.0.3.165   <pending>     80:31380/TCP,443:31390/TCP,31400:31400/TCP,15011:32684/TCP,8060:32261/TCP,853:30607/TCP,15030:30645/TCP,15031:31132/TCP   24h
```
If the `EXTERNAL-IP` value is set, your environment has an external load balancer that you can use for the ingress gateway. If the `EXTERNAL-IP` value is `<none>` (or perpetually `<pending>`), your environment does not provide an external load balancer for the ingress gateway. In this case, you can access the gateway using the service’s [node port](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport).

We are going to use the **node port** to get access from the outside of the cluster (we don't have a loadbalancer).

Use this command to set a port variable :

```bash
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
```

As a result, it shows the port variable:

```console
# echo $INGRESS_PORT
31380
```

Now for the ip address, we should normally use the one from the proxy node. But in this case, the proxy node is also running on the same VM as the master node. 

``` bash
export GATEWAY_URL=$M1IP:$INGRESS_PORT
```

As a result, show the variable:

```console
# echo $GATEWAY_URL
158.176.83.201:31380
```

Now check the following URL:

```bash
curl -s http://${GATEWAY_URL}/productpage | grep -o "<title>.*</title>"
```

Results:

```console
# curl -s http://${GATEWAY_URL}/productpage | grep -o "<title>.*</title>"
<title>Simple Bookstore App</title>
```

Now that we can get the title, let us try this on a browser:

```http
http://${GATEWAY_URL}/productpage
```

Results:

![image-20190326234006819](images/image-20190326234006819-3640006.png)



Try to refresh the page several times, you will see different versions of reviews **randomly** shown in the product page(red stars, black stars, no stars), because I haven’t created any route rule for the Bookinfo application.




# Task 4: Collect Metrics with Prometheus

In this section, you can see how to configure Istio to automatically gather telemetry and create new customized telemetry for services. I will use the Bookinfo application as an example.

Istio can enable Prometheus with a service type of ClusterIP. You can also expose another service of type NodePort and then access Prometheus by running the following command:

```bash
kubectl expose service prometheus --type=NodePort  --name=istio-prometheus-svc --namespace istio-system
```



```bash
export PROMETHEUS_URL=$(kubectl get po -l app=prometheus \
      -n istio-system -o 'jsonpath={.items[0].status.hostIP}'):$(kubectl get svc \
      istio-prometheus-svc -n istio-system -o 'jsonpath={.spec.ports[0].nodePort}')
```

Check the results:

`echo http://${PROMETHEUS_URL}/ `       

Results:
```console
# echo http://${PROMETHEUS_URL}/
http://5.10.96.73:31316/
```

Use the ${PROMETHEUS_URL} to get access to prometheus from a browser.

```http
http://5.10.96.73:31316/
```



Then type `istio_response_bytes_sum` in the first field and click execute button:

![prom1](./images/prom1.png)

Move to the right to see the collected metrics:

![prom2](./images/prom2.png)

If you don't see anything (no metrics yet collected), then retry the following command several times in the terminal:

```bash
curl -o /dev/null -s -w "%{http_code}\n" http://<masterip>:9080/productpage
```



# Task 5: Visualizing Metrics with Grafana

Now I will setup and use the Istio Dashboard to monitor the service mesh traffic. I will use the Bookinfo application as an example.

Similar to Prometheus, Istio enables Grafana with a service type of ClusterIP. You need to expose another service of type NodePort to access Grafana from the external environment by running the following commands:

```bash
kubectl expose service grafana --type=NodePort --name=istio-grafana-svc --namespace istio-system
```

Then export a variable:

```bash
export GRAFANA_URL=$(kubectl get po -l app=grafana -n \
      istio-system -o 'jsonpath={.items[0].status.hostIP}'):$(kubectl get svc \
      istio-grafana-svc -n istio-system -o \
      'jsonpath={.spec.ports[0].nodePort}')
```

Finally:

`echo http://${GRAFANA_URL}/`

Results:
```console
# echo http://${GRAFANA_URL}
http://5.10.96.73:30915
```
Access the Grafana web page from your browser http://${GRAFANA_URL}/.

```http
http://5.10.96.73:30915
```

By default, Istio grafana has some built-in dashboards: Istio Dashboard, Mixer Dashboard and Pilot Dashboard. Istio Mesh Dashboard is an overall view for all service traffic including high-level HTTP requests flowing and metrics about each individual service call, while Mixer Dashboard and Pilot Dashboard are mainly resources usage overview.

Click on the top left HOME button and you should see all the built-in dashboards:

![prom2](./images/grafana1.png)

The Istio Mesh Dashboard resembles the following screenshot (refresh the productpage screen) :

![image-20190707235703747](images/image-20190707235703747-2536623.png)

![image-20190707235919988](images/image-20190707235919988-2536760.png)

 Navigate into the different views:

![image-20190708000005808](images/image-20190708000005808-2536805.png)     



# Congratulations 

You have successfully installed, deployed and customized the **Istio** for an **IBM Cloud Private** cluster.

In this lab, we have enabled Istio on an IBM Cloud Private 3.2.  We also reviewed how to deploy microservice-based application that are managed and secured by Istio. The lab also covered how to manage and monitor microservices with Istio addons such as Prometheus and Grafana.

Istio solves the microservices mesh tangle challenge by injecting a transparent envoy proxy as a sidecar container to application pods. Istio can collect fine-grained metrics and dynamically modify the routing flow without interfering with the original application. This provides a uniform way to connect, secure, manage, and monitor microservices.

For more information about Istio, see https://istio.io/docs/.

----



![icp000](images/icp000.png)