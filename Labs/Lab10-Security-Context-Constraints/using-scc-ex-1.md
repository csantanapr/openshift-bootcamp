# Exercise 1 - Creating and Using Security Context Constraints

In this lab you'll create a new Cassandra database application that requires additional privileges above what the default `restricted` Security Context Constraint (SCC) provides. You'll see how we can create and apply a new Security Context Constraint so that the application can run.

To get started, log into OpenShift using the CLI, as described [here](../Getting-started/log-in-to-openshift.md).

A set of helpful common `oc` commands can be found [here](../Getting-started/oc-commands.md).

Once you're logged in, create a new project for this deployment.

```
$ oc new-project userXX-lab10-scc
```

Replace `userXX` with your user ID or other name.

Create a new local directory for this lab, and change to it.

```
$ mkdir -p Lab10/scc
$ cd Lab10/scc
```

For this lab, you will need to log in to OpenShift with cluster administrative right to enabled the use of SCCs. Please contact the lab administrator to provide you with a valid token to use throughout this lab.

Like a Deployment, a StatefulSet manages Pods that are based on an identical container spec. Unlike a Deployment, a StatefulSet maintains a sticky identity for each of their Pods. These pods are created from the same spec, but are not interchangeable: each has a persistent identifier that it maintains across any rescheduling.

This example will show how a StatefulSet can be coupled with dynamic storage provisioning to fully utilise scalable stateful container applications.

First, we'll create something called a Headless Service. A headless service is a service with a service IP but instead of load-balancing it will return the IPs of our associated Pods. This allows us to interact directly with the Pods instead of a proxy. It's as simple as specifying None for `.spec.clusterIP` and can be utilized with or without selectors. This becomes useful when you want to divert traffic to the individual addresses of each database pod as opposed to letting the service load balance across all pods assigned to the service. For example, for a high-availability database deployment where the leading database needs to keep track of the other cluster members and send data directly to each member pod, using a normal Service will not suffice as the data will be routed to a randomly selected load-balanced pod instead of the target. 

To create the headless service, create a file calledd `headless-svc.yaml`

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: cassandra
  name: cassandra
spec:
  clusterIP: None
  ports:
  - port: 9042
  selector:
    app: cassandra
```

Create the service

```
$ oc create -f headless-svc.yaml
service/cassandra created
```

Next we need to deploy the StatefulSet. Below is the definition for a Cassandra database with 3 replicas. With the use of the `volumeClaimTemplates`, we can define specific storage requirements for the database pods. It also means that each pod in the StatefulSet will create it's own indexed Persistent Volume Claim using a dynamic provisioner available within the cluster. 


Create a new file `cassandra-sts.yaml`

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cassandra
  labels:
    app: cassandra
spec:
  serviceName: cassandra
  replicas: 3
  selector:
    matchLabels:
      app: cassandra
  template:
    metadata:
      labels:
        app: cassandra
    spec:
      terminationGracePeriodSeconds: 1800
      containers:
      - name: cassandra
        image: gcr.io/google-samples/cassandra:v13
        imagePullPolicy: Always
        ports:
        - containerPort: 7000
          name: intra-node
        - containerPort: 7001
          name: tls-intra-node
        - containerPort: 7199
          name: jmx
        - containerPort: 9042
          name: cql
        resources:
          limits:
            cpu: "500m"
            memory: 1Gi
          requests:
            cpu: "500m"
            memory: 1Gi
        securityContext:
          capabilities:
            add:
              - IPC_LOCK
        lifecycle:
          preStop:
            exec:
              command: 
              - /bin/sh
              - -c
              - nodetool drain
        env:
          - name: MAX_HEAP_SIZE
            value: 512M
          - name: HEAP_NEWSIZE
            value: 100M
          - name: CASSANDRA_SEEDS
            value: "cassandra-0.cassandra"
          - name: CASSANDRA_CLUSTER_NAME
            value: "K8Demo"
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
        readinessProbe:
          exec:
            command:
            - /bin/bash
            - -c
            - /ready-probe.sh
          initialDelaySeconds: 15
          timeoutSeconds: 5
        volumeMounts:
        - name: cassandra-data
          mountPath: /cassandra_data
  volumeClaimTemplates:
  - metadata:
      name: cassandra-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: managed-nfs-storage
      resources:
        requests:
          storage: 1Gi
```

Verify the Statefulset was created

```
$ oc get statefulset
NAME        READY   AGE
cassandra   0/3     3s

$ oc get pods
No resources found
```

Even after waiting a short period of time, no pods are ever created. But why?

To find out, we need to start debugging the deployed StatefulSet.

Usually one of the first problems for a database deployment might be that the storage wasn't created.

```
$ oc get pvc
NAME                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
cassandra-data-cassandra-0   Bound    pvc-43aa0899-be91-480a-82d7-31bd1fa39ed7   1Gi        RWO            managed-nfs-storage   83s
```

Looks like the storage was created fine. Let's check the events for the project

```
$ oc get events
LAST SEEN   TYPE      REASON                  OBJECT                                             MESSAGE
75s         Normal    ExternalProvisioning    persistentvolumeclaim/cassandra-data-cassandra-0   waiting for a volume to be created, either by external provisioner "nfs-provisioning" or manually created by system administrator
75s         Normal    Provisioning            persistentvolumeclaim/cassandra-data-cassandra-0   External provisioner is provisioning volume for claim "user99-lab10-scc/cassandra-data-cassandra-0"
75s         Normal    ProvisioningSucceeded   persistentvolumeclaim/cassandra-data-cassandra-0   Successfully provisioned volume pvc-43aa0899-be91-480a-82d7-31bd1fa39ed7
75s         Normal    SuccessfulCreate        statefulset/cassandra                              create Claim cassandra-data-cassandra-0 Pod cassandra-0 in StatefulSet cassandra success
34s         Warning   FailedCreate            statefulset/cassandra                              create Pod cassandra-0 in StatefulSet cassandra failed error: pods "cassandra-0" is forbidden: unable to validate against any security context constraint: [capabilities.add: Invalid value: "IPC_LOCK": capability may not be added]
```

So now we have found the source of the error. Defined in the `cassandra-sts.yaml` is the code
```
securityContext:
  capabilities:
    add:
      - IPC_LOCK
```

So this means that the Cassandra application requires the use of `IPC_LOCK` to be able to run. In any OpenShift cluster the default Security Context Constraint is the `restricted` SCC, which clearly does not provide the right level of privileges we need to run this application.

We can check what privileges the `restricted` SCC does have 

```
$ oc get scc restricted -o yaml
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: true
allowPrivilegedContainer: false
allowedCapabilities: null
apiVersion: security.openshift.io/v1
defaultAddCapabilities: null
fsGroup:
  type: MustRunAs
groups:
- system:authenticated
kind: SecurityContextConstraints
metadata:
  annotations:
    kubernetes.io/description: restricted denies access to all host features and requires
      pods to be run with a UID, and SELinux context that are allocated to the namespace.  This
      is the most restrictive SCC and it is used by default for authenticated users.
  creationTimestamp: "2020-03-14T15:32:48Z"
  generation: 1
  name: restricted
  resourceVersion: "6002"
  selfLink: /apis/security.openshift.io/v1/securitycontextconstraints/restricted
  uid: e42e08f3-9e4a-4338-ba72-f61f5a8063aa
priority: null
readOnlyRootFilesystem: false
requiredDropCapabilities:
- KILL
- MKNOD
- SETUID
- SETGID
runAsUser:
  type: MustRunAsRange
seLinuxContext:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
users: []
volumes:
- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- projected
- secret
```

So a potential solution here is to just add the `IPC_LOCK` feature to the restricted SCC, but that would mean that all other deployment in other projects could also leverage the same feature, and that might not be what we want to do. So the ideal solution is to create a new SCC that extends `restricted` and apply the correct access control to our project.

We can use the `restricted` SCC as a base for the new SCC.

First save the `restricted` SCC locally

```
$ oc get scc restricted -o yaml --export > cassandra-scc.yaml
```

Edit this file and add the section `allowedCapabilities: null` to the following

```
allowedCapabilities:
- IPC_LOCK
```

Also change the name of this SCC from `restricted` to `cassandra-scc-userXX`, replacing XX with your user ID that you have been using in the labs so far. You can also update the `kubernetes.io/description` if you wish.

Save and exit the file.

Apply this file to the cluster

```
$ oc create -f cassandra-scc.yaml
securitycontextconstraints.security.openshift.io/cassandra-scc created
```

Verify the new SCC exists in the list of SCCs available

```
$ oc get scc
NAME                   AGE
anyuid                 108d
cassandra-scc-user99   5s
hostaccess             108d
hostmount-anyuid       108d
hostnetwork            108d
node-exporter          108d
nonroot                108d
privileged             108d
restricted             108d
```

Now we need to apply this SCC to the currrent project. OpenShift makes it very simple to do this with just one command. This command will create all the other required resources that allows the service account within our project (the cassandra deployment only uses the default service account anyway) to consume the SCC. This command takes the following format: `oc adm policy add-scc-to-user [SCC] [SERVICEACCOUNT]`

```
$ oc adm policy add-scc-to-user cassandra-scc-user99 -z default
securitycontextconstraints.security.openshift.io/cassandra-scc-user99 added to: ["system:serviceaccount:user99-lab10-scc:default"]
```

Or alternatively

```
$ oc adm policy add-scc-to-user cassandra-scc-user99 system:serviceaccount:user99-lab10-scc:default
securitycontextconstraints.security.openshift.io/cassandra-scc-user99 added to: ["system:serviceaccount:user99-lab10-scc:default"]
```

Now that the new SCC is in place, we should now see a cassandra pod starting up

```
$ oc get pods
NAME          READY   STATUS             RESTARTS   AGE
cassandra-0   0/1     CrashLoopBackOff   7          12m
```

Ignore the errors as we have met the goals of the lab.

When ready, remove the resources created and the project.

Lab complete.