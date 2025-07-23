Create the Deployment
---------------------

The following [Deployment](https://eksnuggets.substack.com/p/whats-a-deployment-in-kubernetes) manifest will be used to deploy two Pods running version 1.34.1 of BusyBox. We also include a simple command to `execute sleep 3600`, which keeps the container alive for 3,600 seconds:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: simple-deployment-app
  template:
    metadata:
      labels:
        app: simple-deployment-app
    spec:
      containers:
      - name: busybox
        image: busybox:1.34.1
        command:
        - sleep
        - "3600"
```

Use the following command to create the Deployment:

```
kubectl create -f simple-deployment.yaml
```

You will also see the `deployment.apps/simple-deployment created` message in response.

You can also verify the deployment:

```
% kubectl get deployment simple-deployment
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
simple-deployment   2/2     2            2           106s
```

The Deployment is a composite type, and it contains

-   The Deployment itself

-   A ReplicaSet, which is used to maintain the desired state of two Pods per Deployment

-   The Pods

Use the following command to retrieve all resources in the current namespace:

```
% kubectl get all
NAME                                     READY   STATUS    RESTARTS   AGE
pod/simple-deployment-7ffdf4f8c7-652nv   1/1     Running   0          6m5s
pod/simple-deployment-7ffdf4f8c7-89njz   1/1     Running   0          6m5s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   25h

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/simple-deployment   2/2     2            2           6m8s

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/simple-deployment-7ffdf4f8c7   2         2         2       6m8s
```

Modifying your Deployment
-------------------------

We can scale the Deployment with the following command

```
kubectl scale deployment simple-deployment --replicas 3
```

The command will increase the desired number of Pods to three, and in turn, add another Pod.

We can also update the Deployment image with the following command

```
kubectl set image deployment/<deployment-name> <container-name>=<new-image>
```

The command updates a single container's image. You can modify multiple containers' images as well:

```
kubectl set image deployment/myapp frontend=nginx:1.25 backend=python:3.11
```

For our deployment, let's update the BusyBox image:

```
kubectl set image deployment/simple-deployment busybox=busybox:1.35.0
```

Applying that command triggers a rolling update (the default mechanism). You can validate the rollout using the following command:

```
kubectl rollout status deployment/simple-deployment
```

You should see the following output

```
deployment "simple-deployment" successfully rolled out
```

You will see that all the Pods are replaced with the new version. To attach to the Pod shell, use the following command

```
kubectl exec --stdin --tty <pod-id> -- /bib/sh
```

For example:

```
kubectl exec --stdin --tty simple-deployment-679996f5b9-2jmm2 -- /bin/sh
```

Once in the Pod shell, run the following command

```
# busybox | head -1
BusyBox v1.35.0 (2021-12-26 16:56:57 UTC) multi-call binary.
```

Exposing your Deployment
------------------------

### Using a ClusterIP

The Service we will create is a ClusterIP Service. This Service is only visible from inside the cluster. It is exposed on port 80 and mapped to port 9376 on any Pod that has the label of `app=simple-deployment-app`:

```
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: simple-deployment-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
```

We can validate the Service using the `kubectl get service` command:

```
kubectl get service -o wide
```

```
% kubectl get service -o wide
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE     SELECTOR
myapp        ClusterIP   10.100.79.136   <none>        80/TCP    5m10s   app=simple-deployment-app
```

If we look deeper into the Service using the `kubectl describe service` command, we can see the **Endpoints** configuration item:

```
% kubectl describe service myapp
Name:                     myapp
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=simple-deployment-app
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.100.79.136
IPs:                      10.100.79.136
Port:                     <unset>  80/TCP
TargetPort:               9376/TCP
Endpoints:                172.31.12.242:9376,172.31.10.83:9376,172.31.34.20:9376
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```

The Endpoint configuration contains the IP addresses of the Pods that have the label app=simple-deployment-app. We can confirm this with the `kubectl get pods -o wide` command:

```
% kubectl get pods -o wide
NAME                                 READY   STATUS    RESTARTS         AGE   IP              NODE                                         NOMINATED NODE   READINESS GATES
simple-deployment-679996f5b9-2jmm2   1/1     Running   14 (9m11s ago)   14h   172.31.34.20    ip-172-31-46-77.eu-west-1.compute.internal   <none>           <none>
simple-deployment-679996f5b9-gbrpl   1/1     Running   14 (9m7s ago)    14h   172.31.12.242   ip-172-31-6-71.eu-west-1.compute.internal    <none>           <none>
simple-deployment-679996f5b9-zmdlb   1/1     Running   14 (9m12s ago)   14h   172.31.10.83    ip-172-31-6-71.eu-west-1.compute.internal
```

In Kubernetes, Services are discoverable via DNS. The default DNS naming format follows this convention:

```
<service-name>.<namespace>.svc.cluster.local
```

Let's say we have a Service named `myapp` in the default namespace. Its fully qualified domain name (FQDN) would be:

```
myapp.default.svc.cluster.local
```

Here's how each part breaks down:

-   **myapp**: Name of the Service

-   **default**: Kubernetes namespace

-   **svc**: Indicates it's a Service object

-   **cluster.local**: Default domain suffix for the cluster

To ensure that your Kubernetes DNS is working correctly and that the myapp Service is reachable inside the cluster, you can run a DNS lookup from any running Pod:

```
kubectl exec -it <your-running-pod> -- nslookup myapp.default.svc.cluster.local
```

For example:

```
kubectl exec -it simple-deployment-679996f5b9-2jmm2 -- nslookup myapp.default.svc.cluster.local
Server:         10.100.0.10
Address:        10.100.0.10:53

Name:   myapp.default.svc.cluster.local
Address: 10.100.79.136
```

myapp Service is visible in the cluster using the cluster DNS and resolves to `10.100.79.136`, which is the IP address assigned to the **clusterIP**. To expose the Service outside the cluster, we need to use a different type of Service.

### Using a NodePort

Create the following Deployment

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: simple-nginx-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: simple-nginx-app
  template:
    metadata:
      labels:
        app: simple-nginx-app
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

A NodePort Service exposes a static port, between 30000-32768 by default on each worker node in the cluster, and then maps traffic back to port 80 in the configuration shown next:

```
apiVersion: v1
kind: Service
metadata:
  name: myapp-ext
spec:
  type: NodePort
  ports:
  - name: nodeport
    port: 80
    protocol: TCP
  selector:
    app: simple-nginx-app
```

The Service we create is a `NodePort` Service that selects a Pod that has a label of `app=simple-nginx-app`. We can see that the NodePort has been created successfully by running the following command:

```
% kubectl get service
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE

myapp-ext    NodePort    10.100.161.35   <none>        80:32556/TCP   2m2s
```

The IP address shown in the output is the **private IP** assigned to the node.

To test access to the Service from your local machine, use the **public IP** provided by AWS for your EC2 instance.

For example, if your node's public IP is 54.247.224.126, you can run the following curl command from your terminal (**assuming all worker node security groups are configured to allow traffic**):

```
curl 54.247.224.126:32556
```

Or using your browser:

[

![](https://substackcdn.com/image/fetch/$s_!K_SM!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F2be3f0ef-835d-4d82-bad9-ae023f1280f5_2362x526.png)

](https://substackcdn.com/image/fetch/$s_!K_SM!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F2be3f0ef-835d-4d82-bad9-ae023f1280f5_2362x526.png)

### Using an Ingress

[An Ingress builds](https://eksnuggets.substack.com/p/ingress-ingress-controllers-and-services) on top of Services by providing a mechanism to expose HTTP/HTTPS routes, such as `/login`, or `/order`; outside a cluster.

An Ingress is independent of the underlying Services, so a typical use case is to use a single Ingress to provide a central entry point for multiple (micro) services.

To use an Ingress, you need as **Ingress controller**; this is not provided by Kubernetes, so it must be installed. We will use the **NGINX Ingress controller** for this example.

To install the NGINX Ingress controller without cloud/AWS extensions, you can use the following command to deploy the bare-metal controller:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.0/deploy/static/provider/baremetal/deploy.yaml
```

We can verify the new Ingress controller with the following command:

```
kubectl get service ingress-nginx-controller --namespace ingress-nginx
NAME                       TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller   NodePort   10.100.99.47   <none>        80:31865/TCP,443:30235/TCP   4h34m
```

We can see the Ingress controller is exposed as a NodePort Service listening on `31865` for HTTP connections, and `30235` for HTTPS connections.

We can use the previous Service and simply expose a URL on top of the Service using the following manifest:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-web
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: "myweb.cloudbrewery.com"
      http:
        paths:
          - path: /login
            pathType: Prefix
            backend:
              service:
                name: myapp-ext
                port:
                  number: 80
```

The Ingress uses **annotations** to configure the Ingress controller we created previously, with the path rules in the `spec` section.

The rule states that when a request arrives at `myweb.cloudbrewery.com/login`, you need to send it to the `myapp-ext` Service on port 80 and rewrite `/login` to just `/`.

We can test this with the following command, which should return the NGINX welcome page:

```
curl -H 'Host: myweb.cloudbrewery.com' http://54.247.224.126:31865/login
```

### Using an AWS Load Balancer

We have two options to integrate AWS Load Balancers with Ingress controllers:

-   [NodePort + manual Load Balancer](https://eksnuggets.substack.com/i/168946807/option-nodeport-manual-load-balancer), or

-   [Integrated AWS-aware Load Balancer](https://eksnuggets.substack.com/i/168946807/option-integrated-aws-aware-load-balancer)

For production-grade setups in EKS where dynamic scaling and resilience are critical, the second option is best practice.

1.  Delete the existing Ingress and Ingress Controller:

```
kubectl delete -f ingress.yaml
```

```
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.0/deploy/static/provider/baremetal/deploy.yaml
```

1.  Deploy the NGINX controller that is integrated with AWS load balancers:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.0/deploy/static/provider/aws/deploy.yaml
```

From the output below we can see the annotations that will create an AWS Network Load Balancer (NLB) and also the target group for the Ingress controller running in EKS:

```
kubectl describe service ingress-nginx-controller --namespace ingress-nginx
```

```
Name:                     ingress-nginx-controller
Namespace:                ingress-nginx
Labels:                   app.kubernetes.io/component=controller
                          app.kubernetes.io/instance=ingress-nginx
                          app.kubernetes.io/name=ingress-nginx
                          app.kubernetes.io/part-of=ingress-nginx
                          app.kubernetes.io/version=1.2.0
Annotations:              service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp
                          service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: true
                          service.beta.kubernetes.io/aws-load-balancer-type: nlb
Selector:                 app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.100.112.73
IPs:                      10.100.112.73
LoadBalancer Ingress:     addf36edb9cbf4a00b6a0856089a868e-6f2820cbdccbe3e5.elb.eu-west-1.amazonaws.com
Port:                     http  80/TCP
TargetPort:               http/TCP
NodePort:                 http  31314/TCP
Endpoints:                172.31.6.59:80
Port:                     https  443/TCP
TargetPort:               https/TCP
NodePort:                 https  31523/TCP
Endpoints:                172.31.6.59:443
Session Affinity:         None
External Traffic Policy:  Local
Internal Traffic Policy:  Cluster
HealthCheck NodePort:     32751
Events:
  Type    Reason                Age    From                Message
  ----    ------                ----   ----                -------
  Normal  EnsuringLoadBalancer  4m2s   service-controller  Ensuring load balancer
  Normal  EnsuredLoadBalancer   3m59s  service-controller  Ensured load balancer
```

We can see the load balancer we have created using the following command:

```
aws elbv2 describe-load-balancers
```

```
{
    "LoadBalancers": [
        {
            "LoadBalancerArn": "arn:aws:elasticloadbalancing:eu-west-1:123456789098:loadbalancer/net/addf36edb9cbf4a00b6a0856089a868e/6f2820cbdccbe3e5",
            "DNSName": "addf36edb9cbf4a00b6a0856089a868e-6f2820cbdccbe3e5.elb.eu-west-1.amazonaws.com",
            "CanonicalHostedZoneId": "Z2IFOLAFXWLO4F",
            "CreatedTime": "2025-07-23T02:26:55.364000+00:00",
            "LoadBalancerName": "addf36edb9cbf4a00b6a0856089a868e",
            "Scheme": "internet-facing",
            "VpcId": "vpc-09246a6f",
            "State": {
                "Code": "active"
            },
            "Type": "network",
            "AvailabilityZones": [
                {
                    "ZoneName": "eu-west-1a",
                    "SubnetId": "subnet-a336b0c5",
:...skipping...
```

What we've created is a public load balancer (**internet-facing**), meaning the Service is now reachable (through the Ingress controller) from the Internet.

1.  Redeploy the Ingress manifest:

```
kubectl apply -f ingress.yaml
```

1.  Test access using an workstation with Internet access, using the **DNSName** of the load balancer:

```
curl -H 'Host: myweb.cloudbrewery.com' http://addf36edb9cbf4a00b6a0856089a868e-6f2820cbdccbe3e5.elb.eu-west-1.amazonaws.com/login
```

The above command should display the NGINX welcome page.

1.  Delete the Ingress Service

```
kubectl delete -f ingress.yaml
```

1.  Delete the Ingress controller

```
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.0/deploy/static/provider/aws/deploy.yaml
```
