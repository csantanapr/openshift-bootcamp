
# Installation Commands to Install OpenShift on VMWare

This lab will focus on the commands required to install the OpenShift platform on VMWare infrastructure. As mentioned in the lectures, this is purely for informational purposes and will not work in the current environment.

Make new install directory and change to it
```
mkdir ocp-install
cd ocp-install
```

Create a new ssh key for the installation
```
$ ssh-keygen -t rsa -b 4096 -N '' -f ocp_rsa
Generating public/private rsa key pair.
Your identification has been saved in ocp_rsa.
Your public key has been saved in ocp_rsa.pub.
The key fingerprint is:
SHA256:KZ/af3q6q/OAej9Oy+mOHAiSBZ5bl5w9tPoczzBMQYg ibmdemo@workstation
The key's randomart image is:
+---[RSA 4096]----+
|.   . o+         |
|...E..= o        |
| o.. = =         |
| oo . + ..       |
|o..  ..=S        |
| . . .o+*.       |
|    . oo=o       |
|     o.Bo+  o    |
|    ..+=@*BB     |
+----[SHA256]-----+
```

This will output two new files, `ocp_rsa` and `ocp_rsa.pub`.

Create the below example `install-config.yaml` used in a cluster deployed to VMWare infrastructure using vSphere integration capabilities. 
```
cat <<EOF > install-config.yaml
apiVersion: v1
baseDomain: openshift.cloud.lab.com
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: ocp4
platform:
  vsphere:
    vcenter: 172.24.19.21
    username: Administrator@isstlab.org
    password: Passw0rd!
    datacenter: clod-fondly
    defaultDatastore: v7000-storage-13
pullSecret: '{"auths":{"mirror-registry.ocp4.openshift.cloud.lab.com:5001":{"auth": "YWRtaW46YWRtaW5fMjAxOQ==","email": "admin@isstlab.com"}}}'
sshKey: 'SSH_PUBLIC_KEY'
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  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
  kZa6eL8=
  -----END CERTIFICATE-----
imageContentSources:
- mirrors:
  - mirror-registry.ocp4.openshift.cloud.lab.com:5001/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - mirror-registry.ocp4.openshift.cloud.lab.com:5001/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
EOF
```

This has been optimized for pulling images from a private internal image repository where images are mirrored from Red Hat repositories to a local mirror repository. We're not able to do this in this lab so it is just here as a reference.

The `openshift-install` tool is in your `/home/ibmdemo` directory.

Use the `openshift-install` tool to generate a set of manifests from the `install-config.yaml` we created earlier.
```
$ ./openshift-install create manifests --dir=$(pwd)
INFO Consuming Install Config from target directory
WARNING Making control-plane schedulable by setting MastersSchedulable to true for Scheduler cluster settings
```

Now we should be able to use the `openshift-install` tool to generate our ignition configuration files that get copied to our CoreOS nodes.
```
$ ./openshift-install create ignition-configs --dir=$(pwd)
INFO Consuming Master Machines from target directory 
INFO Consuming Worker Machines from target directory 
INFO Consuming Common Manifests from target directory 
INFO Consuming Openshift Manifests from target directory
```

The following files are generated

```
.
├── auth
│   ├── kubeadmin-password
│   └── kubeconfig
├── bootstrap.ign
├── master.ign
├── metadata.json
└── worker.ign
```

Due to the limitations of vSphere as discussed in the Lectures, we need to create an `append-bootstrap.ign` file for the bootstrap node,

```
{
  "ignition": {
    "config": {
      "append": [
        {
          "source": "http://192.168.180.131/ocp4/bootstrap.ign", 
          "verification": {}
        }
      ]
    },
    "timeouts": {},
    "version": "2.1.0"
  },
  "networkd": {},
  "passwd": {},
  "storage": {},
  "systemd": {}
}
```

Here we would typically copy the bootstrap.ign ignition to the webserver, but we cannot do that in this environment so we'll skip this step.

Convert the ignition files into base 64 encoded files

```
$ cat master.ign | base64 -w0 > master.64
$ cat worker.ign | base64 -w0 > worker.64
$ cat bootstrap-append.ign | base64 -w0 > bootstrap-append.64
```

Here we would prepare the CoreOS machines with the ignition files, but we cannot do that in this environment so we'll skip this step.

Now we run the `wait-for bootstrap-complete` command and wait for the bootstrapping process to complete. In this environment it will not work, but this exercise it just to get used to the commands run to deploy an OpenShift cluster.

```
$ ./openshift-install wait-for bootstrap-complete --dir=$(pwd) --log-level=info
```

When bootstrap procecss is complete, we can remove the bootstrap node from the cluster

Now we run the `wait-for install-complete` command and wait for the installation process to complete. In this environment it will not work, but this exercise it just to get used to the commands run to deploy an OpenShift cluster.
```
$ ./openshift-install wait-for install-complete --dir=$(pwd)
```

In a normal scenario, the cluster should have finished installing.

Lab complete.