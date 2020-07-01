# Lab12 - OpenShift Image Registry

In this lab we'll explore how to use the OpenShift Image Registry to push and pull images.

To get started, log into OpenShift using the CLI, as described [here](../Getting-started/log-in-to-openshift.md).

A set of helpful common `oc` commands can be found [here](../Getting-started/oc-commands.md).

Once you're logged in, create a new project for this deployment.

```
$ oc new-project userXX-lab11-res-con
```

Replace `userXX` with your user ID or other name.

Create a new local directory for this lab, and change to it.

```
$ mkdir -p Lab12/image-registry
$ cd Lab12/image-registry
```

Get the image registry endpoint
```
$ HOST=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}’)
```

Log in to the registry
```
$ podman login -u $(oc whoami) -p $(oc whoami -t) --tls-verify=false $HOST 
Login succeeded!
```

Pull an image
```
$ podman pull python:slim-2.7
```

Tag it to match our image registry
```
$ podman tag python:slim-2.7  ${HOST}/myproject/my-python:slim-2.7
```

Push the image to the registry
```
$ podman push ${HOST}/myproject/my-python:slim-2.7
```

Now we can try pulling it again to make sure it's uploaded

```
$ podman pull ${HOST}/myproject/my-python:slim-2.7
```

Lab complete!