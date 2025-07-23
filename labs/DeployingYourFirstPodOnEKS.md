There are several things that happen when you execute the `kubectl run` command, but before we review them, let's look at the manifest being created:

```
kubectl run busybox --image busybox --restart Never --dry-run client -o yaml
```

The command above, shows you the API object/kind being created but will not send it to the Kubernetes API. Here's the output of the command:

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: busybox
  name: busybox
spec:
  containers:
  - args:
    - client
    image: busybox
    name: busybox
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

-   `apiVersion`: Specifies the version of the Kubernetes API you're using for this object. v1 is the stable version for core resources like Pods.

-   `kind`: The type of Kubernetes object you're creating. Here, it's a **Pod**, which is the smallest deployable unit that can run a container.

-   `metadata`: Provides data to uniquely identify the object, including its name, namespace, labels, etc. `creationTimestamp: null` means this YAML is likely auto-generated and hasn't been applied to the cluster yet.

-   `labels`: Key-value pairs used to organise and select resources. This Pod is labeled `run=busybox`, which can be useful for querying or grouping Pods later.

-   `name`: The unique name of this Pod within the namespace. It's called `busybox`.

-   `spec`: Describes the desired behaviour or configuration of the Pod.

-   `args`: Command-line arguments passed to the container. In this case, it's the argument client (possibly for DNS or network tests).

-   `image`: The container image to run in the Pod. `busybox` is a minimal, lightweight Linux utility image often used for testing.

-   Container `name`: Gives a name to the container inside the Pod. It's also called `busybox`, but this is independent of the Pod name.

-   `resources`: Optional CPU and memory limits or requests. This is left empty, meaning no resource constraints are set.

-   `dnsPolicy`: Controls how DNS is configured for the Pod. `ClusterFirst` tells the Pod to use the cluster DNS service first when resolving names.

-   `restartPolicy`: Specifies when the Pod should be restarted. `Never` means it won't be restarted automatically, even if it fails.

-   `status`: Reports the current status of the Pod. Empty here, since the Pod hasn't been run yet.

```
kubectl run -it busybox --image busybox --restart Never
```

1.  The `kubectl run` command will create the manifest for the Pod and submit it to the Kubernetes API.

2.  The API server will persist the Pod specification.

3.  The scheduler will pick up the new Pod specification, review it, and through a process of filtering and scoring, select a worker node to deploy the resource onto and mark the Pod spec for this node.

4.  On the node, the kubelet agent is monitoring the cluster datastore, etcd, and if a new Pod specification is found, the specification is used to create the Pod on the node.

5.  Once the Pod has started, your kubectl session will attach to the Pod (as we specified with the `-it` flag). You will now be able to use Linux commands to interact with your Pod. You can leave the session by typing `exit`.

Once you exit the session, you can verify the Pod status as follows:

```
% kubectl get pods
NAME      READY   STATUS      RESTARTS   AGE
busybox   0/1     Completed   0          9m9s
```

The Pod status is *Completed*, because we specified `restartPolicy: Never`, so once the interactive session has terminated, the container is no longer accessible.

In the *Deploying a Pod using a Kubernetes Deployment*, we extend this concept of a Pod into a Deployment.

You can delete the Pod using the following command:

```
% kubectl delete pod busybox
pod "busybox" deleted
```
