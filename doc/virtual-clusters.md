#### DEPLOYING VIRTUAL CLUSTERS

The "deploy/example_clusters" directory contains examples of YAML files that can be used to create virtual clusters that instantiate the defined app types. Currently these virtual clusters must be created in the same namespace as the KubeDirector deployment (a restriction that should be relaxed in later development).

For example, this would create an instance of a virtual cluster from the spark221e2 app type:
```bash
    kubectl create -f deploy/example_clusters/cr-cluster-spark221e2.yaml
```
or
```bash
    kubectl create -f deploy/example_clusters/cr-cluster-spark221e2-gke.yaml
```

Those two example files illustrate a couple of important differences in how you choose to deploy a cluster, depending on your K8s environment:
1. The "serviceType" property at the top level of the virtual cluster spec, which defaults to "NodePort" if not specified. For GKE environments you should almost always include this property and set its value to "LoadBalancer". This indicates to KubeDirector that the member services of the virtual cluster should be exposed as LoadBalancer services rather than NodePort.
2. Persistent storage for roles. If you choose to request this through the "storage" role property, make sure that the nested "storageClass" property is set to a persistent storage class that is valid in your K8s environment (verify using "kubectl get storageclasses").

For more details, see the KubeDirector wiki for a [complete spec of the KubeDirectorCluster resource type](https://github.com/bluek8s/kubedirector/wiki/Type-Definitions-for-KubeDirectorCluster).

#### INSPECTING

The virtual cluster will be represented by a resource of type KubeDirectorCluster, with the name that was indicated inside the YAML file used to create it. So for example the virtual cluster created from cr-cluster-spark221e2.yaml has the name "spark-instance", and after creating it you could observe its status using kubectl:
```bash
    kubectl get -o yaml KubeDirectorCluster spark-instance
```

To guarantee that services provided by this virtual cluster are available, wait for the virtual cluster status to indicate that its overall "state" (top-level property of the status object) has a value of "ready". The first time a virtual cluster of a given app type is created, it may take some minutes to reach "ready" state, as the relevant Docker image must be downloaded and imported.

The resource's status will also show you which standard K8s elements make up the virtual cluster (statefulsets, pods, services, and persistent volume claims). You can use kubectl to examine those in turn. Services are particularly useful to examine as they will describe which K8s node ports or loadbalancer ports are mapped to service endpoints on members of the virtual cluster.

To get a report on all services related to a specific virtual cluster, you can use a form of "kubectl get" that matches against a value of the "kubedirectorcluster" label. For example if your virtual cluster is named "spark-instance", you could perform this query:
```bash
    kubectl get services -l kubedirectorcluster=spark-instance
```

Below is a line from the output of such a query. It shows that port 8080 (the Spark master Web dashboard) on the controller host of a virtual Spark cluster is available on port 30311 of any of the K8s nodes:
```bash
svc-spark-instance-controller-zzwt7-0   NodePort    10.107.133.249   <none>        22:31394/TCP,8080:30311/TCP,7077:30106/TCP,8081:30499/TCP   12m
```

As another example, below is a line associated with a different virtual cluster, running in GKE and configured to request a serviceType of LoadBalancer. In this case the Spark master Web dashboard is available through the load-balancer IP 35.227.159.141. The port exposed on the load balancer will be the same as the native container port, 8080.
```bash
svc-spark-instance-controller-zxqjf-0   LoadBalancer   10.55.246.39    35.227.159.141   22:31739/TCP,8080:30858/TCP,7077:30969/TCP,8081:32297/TCP   3m
```

A few notes about using the example applications:
* In both the Spark and TensorFlow examples, the password for Jupyter is "admin123".
* The deployed containers will be running sshd, but they will not initially have any login-capable accounts. For container access as a root user, use "kubectl exec" along with the podname. E.g. "kubectl exec -it cassandra-lb-persistent-seed-pr962-0 bash". From there you can reconfigure sshd if you wish.
* Clickable links to other IPs within the Spark web dashboard will not work, because Spark is not aware of NodePort ports or LoadBalancer IPs.

#### RESIZING

You can edit the resource YAML file to add or remove a role, or increase/decrease the number of members in a role. Then you can apply the changed file:
```bash
    kubectl apply -f deploy/example_clusters/cr-cluster-spark221e2.yaml
```

Depending on the app definition, some resize operations may not be allowed for some roles. For example you will not be allowed to remove a Spark controller or have fewer than two Cassandra seeds. In these cases the resize attempt will be immediately rejected with an explanation.

If a resize that grows the virtual cluster is accepted, but the status shows that some members are staying in create_pending state indefinitely, you may have requested more resources than your K8s nodes can provide. Use kubectl to examine the associated pods, see if they are stuck in Pending status, and what Events they are experiencing. If they appear to be permanently blocked without available resources, you will want to downsize or remove virtual cluster roles so that they no longer request as many members.

Reconfigurations other than resizing are not yet supported.

#### DELETING

Note that deletion of any KubeDirector-managed virtual clusters must be performed while KubeDirector is running. Manual steps can be taken to force their deletion if KubeDirector is absent (see the end of this doc), but in the normal course of things virtual cluster deletion is gated on approval from KubeDirector.

A virtual cluster can be deleted using "kubectl delete" e.g.
```bash
    kubectl delete -f deploy/example_clusters/cr-cluster-spark221e2.yaml
```

Or alternately by type and name:
```bash
    kubectl delete KubeDirectorCluster spark-instance
```

Deleting the virtual cluster resource will automatically delete all resources that compose the virtual cluster.

If you ever want to delete all KubeDirector-managed virtual clusters in the current namespace, you can do:

```bash
    kubectl delete KubeDirectorCluster --all
```

#### FORCE DELETING

It may happen that a virtual cluster refuses to go away, either on explicit manual deletion or during "make teardown" (which will block the teardown process). This may be a sign that KubeDirector has stopped running or the KubeDirector deployment/pod has been deleted.

You can use "kubectl logs" on the KubeDirector pod to see if it stopped (and why). If you are working on KubeDirector development yourself, it may be possible to rescue KubeDirector at this point. However if you simply need to allow the virtual clusters to be deleted, without restoring KubeDirector, you need to remove the "finalizers" from each such cluster resource. Below is a kubectl command that could be used to clear the finalizers from a virtual cluster named "spark-instance":
```bash
kubectl patch kubedirectorcluster spark-instance --type json --patch '[{"op": "remove", "path": "/metadata/finalizers"}]'
```

Once the finalizers are removed, any already-requested deletion should then complete.
