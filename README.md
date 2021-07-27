# odf-ocp-tips

ocs-ocp-i3en is a collection of documentation and configuration for installing and configuring OpenShift Data Foundations ODF (previously OpenShift Container Storage - OCS) on AWS i3en instances.

## Pre-requisites
Infrastructure and networking in place to support an OpenShift v4 installation.

A copy of this repository.
```bash
git clone https://github.com/therevoman/odf-ocp-tips.git
```

Setting an environment variable for the platform type.
```bash
export OCP_PLATFORM_TYPE=aws
```

## Installation

Use the openshift-install binary [openshift-install](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-linux.tar.gz) to install OpenShift.

Installation documentation for all platforms is available at [docs.openshift.com](https://docs.openshift.com/container-platform/latest/installing/index.html)

```bash
openshift-install create cluster
```

Once the cluster is created, create a MachineSet (or logical set of machines) to host the ODF cluster.
Complete documentation can be found in one of the [online guide](https://docs.openshift.com/container-platform/latest/storage/persistent_storage/persistent-storage-ocs.html) for your platform.

__Tip:__ see My [Quick Steps for AWS](#quick-steps-for-aws) for an example.

#### NOTE: I am a firm believer of having nodes dedicated to ODF [see explanation elsewhere]().


Make sure to label the nodes as infrastructure and openshift-storage nodes.  
__Tip:__ having he value portion of the defined as the empty string `""` is critically important.

```bash
oc label node <node> node-role.kubernetes.io/infra=""
oc label node <node> cluster.ocs.openshift.io/openshift-storage=""
```

Also taint the nodes according to [this documentation](https://red-hat-storage.github.io/ocs-training/training/infra-nodes/ocs4-infra-nodes.html#_manual_creation_of_infrastructure_nodes)

```bash
oc adm taint node <node> node.ocs.openshift.io/storage="true":NoSchedule
```

Because I believe in metrics bubbling up to the platform, pre-create the namespaces and mark it for the metrics operator.
```bash
oc create namespace openshift-local-storage
oc label namespace openshift-local-storage "openshift.io/cluster-monitoring=true"
oc create namespace openshift-storage
oc label namespace openshift-storage "openshift.io/cluster-monitoring=true"
```

Here is where you choose your own adventure.  Install the `Local Storage` Operator and the `OpenShift Container Storage` Operator either from the web ui or the command line.  Reference the links mentioned above.

Once the Local Storage Operator installed come back the CLI and create the auto-discovery CR.  This custom file includes the tolerations and node selectors needed to scan for disks only on the ODF nodes.
```bash
oc create -f allplatforms/localvolumediscovery.yaml
```

Monitor the events for this object and verify the appropriate nodes are being scanned.
```bash
oc describe -n openshift-local-storage LocalVolumeDiscovery/auto-discover-devices
```

Now its time to have the operator create PV's for each of the volumes discovered and make them available in a StorageClass.
```bash
oc create -f allplatforms/localvolumeset.yaml
``` 

Now watch as the PersistentVolumes get created
```bash
oc get pv -w
```

Heading into our final steps we have our cluster up with nodes dedicated for consumption by ODF.  
Next modify the StorageCluster CR.
```bash
vim allplatforms/storagecluster.yaml
```

Finally create the StorageCluster and watch for the Phase to change to `Ready`
```bash
oc create -f allplatforms/storagecluster.yaml
oc get StorageCluster ocs-storagecluster -n openshift-storage -w
```



## Quick Steps for AWS:
1) Modify the 3 yaml files to match your region and node type.
```bash
vim aws/infra-machine-set-us-east-2?.yaml
```
2) Apply the MachineSet configuration to your cluster
```bash
oc create -f aws/infra-machine-set-us-east-2a.yaml
oc create -f aws/infra-machine-set-us-east-2b.yaml
oc create -f aws/infra-machine-set-us-east-2c.yaml
```
3) Scale to the number of nodes desired (my case its 3)
```bash
oc scale machineset --replicas=1 <your machineset name>
```
4) Wait for the nodes to show up
```bash
oc get nodes -w
# or 
oc get machines -o wide -w
```

## Roadmap
This is current for OCP 4.7 and ODF 4.7 and specifically targets an AWS IPI style installation.  Adaptations are welcome but not currently planned


## Contributing
Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

Please make sure to update tests as appropriate.

## License
[MIT](https://choosealicense.com/licenses/mit/)
