# Running Rook / Ceph on DigitalOcean DOKS (Managed Kubernetes)

## About this guide

In this guide we will deploy a DigitalOcean Managed Kubernetes cluster, create two Node Pools, one for our standard applications and one dedicated to our Rook / Ceph storage nodes. We'll then deploy a Rook / Ceph cluster and give some working examples of consuming RWO Block and RWX Filesystem storage.

### About Rook

> Rook is an open source cloud-native storage orchestrator, providing the platform, framework, and support for Ceph storage to natively integrate with cloud-native environments.

> Ceph is a distributed storage system that provides file, block and object storage and is deployed in large scale production clusters.

> Rook automates deployment and management of Ceph to provide self-managing, self-scaling, and self-healing storage services. The Rook operator does this by building on Kubernetes resources to deploy, configure, provision, scale, upgrade, and monitor Ceph.

> The Ceph operator was declared stable in December 2018 in the Rook v0.9 release, providing a production storage platform for many years. Rook is hosted by the Cloud Native Computing Foundation (CNCF) as a graduated level project.

### About DigitalOcean DOKS

> DigitalOcean Kubernetes (DOKS) is a managed Kubernetes service that lets you deploy Kubernetes clusters without the complexities of handling the control plane and containerized infrastructure. Clusters are compatible with standard Kubernetes toolchains and integrate natively with DigitalOcean Load Balancers and volumes.

## Deploy a DigitalOcean DOKS Cluster

Import: https://docs.digitalocean.com/products/kubernetes/how-to/create-clusters/

### How to create a Kubernetes cluster using the DigitalOcean CLI

To create a Kubernetes cluster via the command-line, follow these steps:

1.  [Install `doctl`](https://docs.digitalocean.com/reference/doctl/how-to/install/), the DigitalOcean command-line tool.

2.  [Create a personal access token](https://docs.digitalocean.com/reference/api/create-personal-access-token/), and save it for use with `doctl`.

3.  Use the token to grant `doctl` access to your DigitalOcean account.

    ```
    doctl auth init
    ```

4.  Finally, create a Kubernetes cluster with `doctl kubernetes cluster create`.

    ```
    doctl kubernetes cluster create doks-shark-1 --auto-upgrade=true --ha=true --node-pool="name=pool-apps;size=s-4vcpu-8gb-amd;count=3" --region=lon1 --surge-upgrade=true
    ```

* `doks-shark-1` is our cluster name
* We are creating one node pools, 1x for our regular workloads and in the next step we will create 1x dedicated to our storage nodes.
* The region is London, surge-upgrades are enabled, HA is enabled, auto-upgrade is enabled.
* You'll want to [read the usage docs for more details](https://docs.digitalocean.com/reference/doctl/reference/kubernetes/cluster/create/)

## Add a node pool dedicated to our storage (OSD) nodes

1. Grab our DOKS cluster ID

```
doctl kubernetes cluster list
```

```
ID                                      Name              Region    Version        Auto Upgrade    Status     Node Pools
35eabc79-a6a0-4559-8857-38ed09630bfc    doks-shark-1      lon1      1.26.3-do.0    true            running    pool-apps pool-storage
```

2. Create our new node pool
```
doctl kubernetes cluster node-pool create 35eabc79-a6a0-4559-8857-38ed09630bfc --count 3 --name pool-storage --size s-4vcpu-8gb-amd --taint storage-node=true:NoSchedule
```

* We add taints to our storage nodes to make sure that no pods will be scheduled on this node pool as long as we explicitly tolerate it. These nodes are exclusively for Rook / Ceph OSD storage pods.

## Install Rook

Let’s start installing Rook by cloning the repository from GitHub. We'll use a combination of the examples provided in this repo and examples below.

```
git clone https://github.com/jkpedo/rook.git
git checkout doks
```

### Add Common Resources

```
kubectl create -f deploy/examples/crds.yaml -f deploy/examples/common.yaml
```

### Deploy the Rook Operator

```
kubectl create -f deploy/examples/operator.yaml

# verify the rook-ceph-operator is in the `Running` state before proceeding
kubectl get pods -n rook-ceph -l app=rook-ceph-operator
```

To watch the operator you can tail its logs

```
kubectl logs -n rook-ceph -f -l app=rook-ceph-operator
```

## Create a Rook / Ceph Cluster

In this example, we will leverage the `do-block-storage` StorageClass that will be used to request DigitalOcean Volumes which will in turn be dynamically attached to our storage nodes.

```
kubectl create -f deploy/examples/cluster-on-pvc-doks.yaml
```

After a few minutes, you will see some pods running in the rook-ceph namespace. You should be able to see the following pods once they are all running. The number of osd pods will depend on the number of nodes in the cluster and the number of devices configured. It is expected that one OSD will be created per node.

```
kubectl -n rook-ceph get pods

NAME                                                 READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-provisioner-d77bb49c6-n5tgs         5/5     Running     0          140s
csi-cephfsplugin-provisioner-d77bb49c6-v9rvn         5/5     Running     0          140s
csi-cephfsplugin-rthrp                               3/3     Running     0          140s
csi-rbdplugin-hbsm7                                  3/3     Running     0          140s
csi-rbdplugin-provisioner-5b5cd64fd-nvk6c            6/6     Running     0          140s
csi-rbdplugin-provisioner-5b5cd64fd-q7bxl            6/6     Running     0          140s
rook-ceph-crashcollector-minikube-5b57b7c5d4-hfldl   1/1     Running     0          105s
rook-ceph-mgr-a-64cd7cdf54-j8b5p                     1/1     Running     0          77s
rook-ceph-mon-a-694bb7987d-fp9w7                     1/1     Running     0          105s
rook-ceph-mon-b-856fdd5cb9-5h2qk                     1/1     Running     0          94s
rook-ceph-mon-c-57545897fc-j576h                     1/1     Running     0          85s
rook-ceph-operator-85f5b946bd-s8grz                  1/1     Running     0          92m
rook-ceph-osd-0-6bb747b6c5-lnvb6                     1/1     Running     0          23s
rook-ceph-osd-1-7f67f9646d-44p7v                     1/1     Running     0          24s
rook-ceph-osd-2-6cd4b776ff-v4d68                     1/1     Running     0          25s
rook-ceph-osd-prepare-node1-vx2rz                    0/2     Completed   0          60s
rook-ceph-osd-prepare-node2-ab3fd                    0/2     Completed   0          60s
rook-ceph-osd-prepare-node3-w4xyz                    0/2     Completed   0          60s
```

### Verify the Ceph Cluster status or health

You can get or describe the status of your cephcluster in two ways:

```
kubectl get cephclusters.ceph.rook.io  -n rook-ceph
```

```
NAME        DATADIRHOSTPATH   MONCOUNT   AGE   PHASE   MESSAGE                        HEALTH        EXTERNAL   FSID
rook-ceph   /var/lib/rook     3          96m   Ready   Cluster created successfully   HEALTH_OK              e29bbbb1-0abb-4f3d-812c-3527f9e3bfdd
```

Or, to verify that the cluster is in a healthy state, connect to the Rook toolbox and run the ceph status command.

Launch the rook-ceph-tools pod:

```
kubectl create -f deploy/examples/toolbox.yaml
```

Once the rook-ceph-tools pod is running, you can connect to it with:

```
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

All available tools in the toolbox are ready for your troubleshooting needs.

Example:

```
ceph status
ceph osd status
ceph df
rados df
```

#### Deploy Dashboard

Ceph has a dashboard in which you can view the status of your cluster.

```
kubectl -n rook-ceph port-forward svc/rook-ceph-mgr-dashboard 8443:8443
```

The default username is `admin`, to set your own password:

```
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
echo 'your-password-here' > /tmp/test
ceph dashboard ac-user-create admin -i /tmp/test administrator
```

## Configure Rook / Ceph Storage

Rook can expose three different types of storage to your Kubernetes cluster for consumption by your workloads. They are:

* Block: Create block storage to be consumed by a pod (RWO)
* Shared Filesystem: Create a filesystem to be shared across multiple pods (RWX)
* Object: Create an object store that is accessible inside or outside the Kubernetes cluster

For the purposes of this guide we will deploy Block (RWO) and Shared (RWX) storage.

### Block Storage

Block storage allows a single pod to mount storage (RWO)

#### Create the Block StorageClass (RWO)

Before Rook can provision storage, a StorageClass and [CephBlockPool CRD](https://rook.io/docs/rook/v1.11/CRDs/Block-Storage/ceph-block-pool-crd/) need to be created. This will allow Kubernetes to interoperate with Rook when provisioning persistent volumes.

```
kubectl create -f deploy/examples/csi/rbd/storageclass-doks.yaml
```

### Filesystem Storage

A filesystem storage (also named shared filesystem) can be mounted with read/write permission from multiple pods. This may be useful for applications which can be clustered using a shared filesystem.

#### Create the Filesystem

In this example we create the metadata pool with replication of three and a single data pool with replication of three. For more options, see the documentation on [creating shared filesystems](https://rook.io/docs/rook/v1.11/CRDs/Shared-Filesystem/ceph-filesystem-crd/).

```
kubectl create -f deploy/examples/filesystem-doks.yaml
```

To confirm the filesystem is configured, wait for the mds pods to start:

```
kubectl -n rook-ceph get pod -l app=rook-ceph-mds

NAME                                    READY   STATUS    RESTARTS   AGE
rook-ceph-mds-myfs-a-57b8fbb74d-h8mhl   2/2     Running   0          108m
rook-ceph-mds-myfs-b-644bfd856f-c99x4   2/2     Running   0          108m
```

#### Create the Filesystem StorageClass (RWX)

Before Rook can start provisioning storage, a StorageClass needs to be created based on the filesystem. This is needed for Kubernetes to interoperate with the CSI driver to create persistent volumes.

```
kubectl create -f deploy/examples/csi/cephfs/storageclass-doks.yaml
```

## Consuming our new storage cluster

Now we will create some Block Storage (RWO) PVCs and some Filesystem (RWX) PVCs and consume them using some test pods.

### Create some Block Storage (RWO) PVCs

```
kubectl apply -f deploy/examples/doks/pvc-ceph-block-1.yaml
kubectl apply -f deploy/examples/doks/pvc-ceph-block-2.yaml
```

### Create a Filesystem Storage (RWX) PVC

```
kubectl apply -f deploy/examples/doks/pvc-ceph-fs-1.yaml
```

#### Check the status of our new PVCs

```
kubectl get pvc -l test=ceph
```

```
NAME                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
ceph-block-pvc-1        Bound    pvc-4901b00c-ff3c-4677-9b48-a0ae914471f8   128Mi      RWO            rook-ceph-block   99m
ceph-block-pvc-2        Bound    pvc-b51b3549-bfcb-40b3-8248-3825d0510907   128Mi      RWO            rook-ceph-block   99m
ceph-filesystem-pvc     Bound    pvc-984cbc60-63cc-4b63-a7fd-2ac09254ef98   128Mi      RWX            rook-cephfs       4s
```

## Create some Pods that consume the PVCs

Now we'll create two pods.

Pod `ceph-test-1` will mount PVC `ceph-block-pvc-1` as RWO at mountPath /rwo.

Pod `ceph-test-2` will mount PVC `ceph-block-pvc-2` as RWO at mountPath /rwo.

Both pods mount PVC `ceph-filesystem-pvc` as RWX at mountPath /rwx

```
kubectl apply -f deploy/examples/doks/pod-rwo-rwx-1.yaml
kubectl apply -f deploy/examples/doks/pod-rwo-rwx-2.yaml
```

### Test the ReadWriteMany RWX storage

Lets test our RWX storage by creating a file from Pod 1 and reading it from Pod 2

```
kubectl exec -it pod/ceph-test-1 -- touch /rwx/test
```

```
kubectl exec -it pod/ceph-test-2 -- ls -l /rwx
total 0
-rw-r--r--    1 root     root             0 Apr 26 09:24 test
```

# Conclusion

*Need to re-write this part*

We have created a Ceph storage cluster on a DigitalOcean DOKS cluster that uses PVCs to manage storage.
The usage of volume mounts in your deployments with Ceph is now super-fast and rock-solid, because we do not have to attach physical disks to our worker nodes anymore. We just use the ones we have created during Rook cluster provisioning (remember these four 100GB disks?)! We minimized the amount of “physical attach/detach” actions on our nodes.
