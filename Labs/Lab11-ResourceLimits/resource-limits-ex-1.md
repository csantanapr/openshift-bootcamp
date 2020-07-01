# Exercise 1 - Using Resource Constraints

In this lab you'll create and enforce a set of quotas on a project to prevent the applications deployed to that project from breaching a set of resource constraints that we will specify.

To get started, log into OpenShift using the CLI, as described [here](../Getting-started/log-in-to-openshift.md).

A set of helpful common `oc` commands can be found [here](../Getting-started/oc-commands.md).

Once you're logged in, create a new project for this deployment.

```
$ oc new-project userXX-lab11-res-con
```

Replace `userXX` with your user ID or other name.

Create a new local directory for this lab, and change to it.

```
$ mkdir -p Lab11/resource-constraints
$ cd Lab11/resource-constraints
```

For this lab, you will need to log in to OpenShift with cluster administrative right to enabled the use of SCCs. Please contact the lab administrator to provide you with a valid token to use throughout this lab.

After a resource quota for a project is first created, the project restricts the ability to create any new resources that may violate a quota constraint until it has calculated updated usage statistics.

After a quota is created and usage statistics are updated, the project accepts the creation of new content. When you create or modify resources, your quota usage is incremented immediately upon the request to create or modify the resource.

When you delete a resource, your quota use is decremented during the next full recalculation of quota statistics for the project. A configurable amount of time determines how long it takes to reduce quota usage statistics to their current observed system value.

If project modifications exceed a quota usage limit, the server denies the action, and an appropriate error message is returned to the user explaining the quota constraint violated, and what their currently observed usage statistics are in the system.

For this example, we're going to enforce a the following constraints

- The total number of pods in a non-terminal state that can exist in the project.
- Across all pods in a non-terminal state, the sum of CPU requests cannot exceed 1 core.
- Across all pods in a non-terminal state, the sum of memory requests cannot exceed 1Gi.
- Across all pods in a non-terminal state, the sum of ephemeral storage requests cannot exceed 2Gi.
- Across all pods in a non-terminal state, the sum of CPU limits cannot exceed 2 cores.
- Across all pods in a non-terminal state, the sum of memory limits cannot exceed 2Gi.
- Across all pods in a non-terminal state, the sum of ephemeral storage limits cannot exceed 4Gi.
 
 We can do this with the following specification

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    pods: "4"
    requests.cpu: "1" 
    requests.memory: 1Gi 
    requests.ephemeral-storage: 2Gi 
    limits.cpu: "2" 
    limits.memory: 2Gi
```

Save this to a file called `compute-resources.yaml`, then create the ResourceQuota using `oc create -f compute-resources.yaml`

```
$ oc create -f compute-resources.yaml
resourcequota/compute-resources created
```

Now let's see what happens when we try to create a few `hello-openshift` deployments. 

Create a `deployment.yaml` from the following content

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example
  namespace: user99-lab11-res-con
spec:
  selector:
    matchLabels:
      app: hello-openshift
  replicas: 2
  template:
    metadata:
      labels:
        app: hello-openshift
    spec:
      containers:
        - name: hello-openshift
          image: openshift/hello-openshift
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 200m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

Deploy it to OpenShift

```
$ oc create -f deployment.yaml
deployment.apps/example created
```

Create a copy of that file called `deployment2.yaml`, this time editing the `name` field to something different to avoid name clashes and increase the number of replicas from 2 to 3, and deploy it again.

```
$ oc create -f deployment2.yaml
deployment.apps/example2 created
```

Now check the pods that are created

```
$ oc get pods
NAME                       READY   STATUS    RESTARTS   AGE
example-f4894b6b8-5kxqd    1/1     Running   0          97s
example-f4894b6b8-ptnh6    1/1     Running   0          110s
example2-f4894b6b8-lwz88   1/1     Running   0          52s
example2-f4894b6b8-z8bxm   1/1     Running   0          52s
```

Despite creating deployments with 5 replicas defined, the scheduler will not create any new pods until we are under quota. Whilst it's not completely obvious at first, since we're expecting 5 pods and only see 4, we can check the events to look for errors.

```
$ oc get events
LAST SEEN   TYPE      REASON              OBJECT                          MESSAGE
2m23s       Warning   FailedCreate        replicaset/example2-f4894b6b8   Error creating: pods "example2-f4894b6b8-pjz76" is forbidden: exceeded quota: compute-resources, requested: limits.cpu=500m,limits.memory=512Mi,pods=1, used: limits.cpu=2,limits.memory=2Gi,pods=4, limited: limits.cpu=2,limits.memory=2Gi,pods=4
```

So now we know why we don't have 5 pods, we have exceeded the number of pods. This same process can apply to a range of different resource types. More information andd examples can be found at https://docs.openshift.com/container-platform/4.3/applications/quotas/quotas-setting-per-project.html

Lab complete.