```
[ETCD](https://rudimartinsen.com/2020/12/30/backup-restore-etcd/#backing-up-the-etcd-cluster)

Accessing the etcd cluster
To interact with the etcd cluster we use the etcdctl binary which can be downloaded here. When run in Kubernetes we can access it through the etcd pods running on the control plane nodes

Access etcd cluster through a pod
kubectl -n kube-system get pods
BASH
List pods of kube-system
List pods of kube-system

Now, let's run the etcdctl command from inside one of those pods to retrieve the etcd version running

kubectl -n kube-system exec <etcd-pod-name> -- sh -c "etcdctl version"
BASH
Run etcdctl from a pod
Run etcdctl from a pod

If we want to pull data from the cluster we have to authenticate, normally through certificates as mentioned before (as of now etcd authentication is not supported by Kubernetes).

First let's check the configuration on one of the control plane nodes. The (initial) configuration is stored in a yaml file, etc/kubernetes/manifests/etcd.yaml

Etcd yaml manifest
Etcd yaml manifest

A couple of things to note here. First the advertise-client-url which can be used from outside of the node, and then the certificate paths which we need for authenticating

Now let's list the members of the etcd cluster

kubectl -n kube-system exec <etcd-pod-name> -- sh -c "ETCDCTL_API=3 etcdctl member list --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key"
BASH
Etcd member list
Etcd member list

The normal (or simple) output writes the following columns: "ID", "Status", "Name", "Peer Addrs", "Client Addrs", "Is Learner". The learner column/state was introduced in version 3.4 (the current version) and if this is false it means that everything is ok. If a node has a state of true in that column, i.e. it is a "learner", it means that it is not a full member, only a standby node that needs to catch up with the leader.

To get a bit more details, such as the cluster leader and the version we can use the endpoint status command. Note that if you're not specifying more endpoints you'll only get the local. In the second output I've specified the endpoint IP of all three nodes found in the member list as well as used the --write-output=table flag to get a nice output

kubectl -n kube-system exec <etcd-pod-name> -- sh -c "ETCDCTL_API=3 etcdctl endpoint status --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key"

kubectl -n kube-system exec <etcd-pod-name> -- sh -c "ETCDCTL_API=3 etcdctl endpoint status --write-out=table --endpoints=https://<IP1>:2379,https://<IP2>:2379,https://<IP3>:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key"
BASH
Endpoint status
Endpoint status

So from this output we learn that the node with the 192.168.151.13 IP is the etcd leader, we're using version 3.4.13 and that there's no errors on the members.

Access etcd cluster from outside of Kubernetes
Now, let's try to access the etcd cluster from outside of Kubernetes. Actually I'll use one of the control plane nodes, but I'll download the etcdctl binaries and run it from the OS of the node. This is something you probably shouldn't do in a production environment!

I've downloaded the binaries from the etcd github repo, make sure you use a compatible version to what is running in your cluster.

After unpacking the binaries and moving them to the /usr/local/bin directory I can run etcdctl from my node directly without going through a pod

wget https://github.com/etcd-io/etcd/releases/download/v3.4.14/etcd-v3.4.14-linux-amd64.tar.gz
tar xvf etcd-v3.4.14-linux-amd64.tar.gz
sudo mv etcd-v3.4.14-linux-amd64/etcd* /usr/local/bin

etcdctl version
BASH
Running etcdctl from os
Running etcdctl from os

Now let's try to access the cluster with the certificates as before

sudo ETCDCTL_API=3 etcdctl endpoint status --write-out=table --endpoints=https://<IP1>:2379,https://<IP2>:2379,https://<IP3>:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key
BASH
Accessing etcd from outside of Kubernetes
Accessing etcd from outside of Kubernetes

Backing up the etcd cluster
Now, let's finally get to performing a backup of the etcd cluster which is one of the objectives in the CKA exam.

There is two ways to backup the etcd cluster, the built-in snapshot and a volume snapshot.

Oftentimes you'll run etcd on a storage volume that supports backup hence you can just take a snapshot of the storage volume directly.

If you want to use the built-in snapshot, which I suspect is the case in the CKA exam, we can take a snapshot from a member with the etcdctl snapshot save snapshotdb command or by copying the member/snap/db file from a etcd data directory not currently in use by an etcd process. I'll do the snapshot through etcdctl

sudo ETCDCTL_API=3 etcdctl snapshot save snapshotdb --endpoints=https://<IP>:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key
BASH
Etcdctl snapshot creation
Etcdctl snapshot creation

Snapshot taken, let's run the snapshot status snapshotdb command to verify the snapshot

sudo ETCDCTL_API=3 etcdctl snapshot status snapshotdb --endpoints=https://<IP>:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key
BASH
Etcdctl snapshot creation
Etcdctl snapshot creation

I'll quickly perform a new snapshot and do another status command to verify that the snapshot is updated

sudo ETCDCTL_API=3 etcdctl snapshot save snapshotdb --endpoints=https://<IP>:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key

sudo ETCDCTL_API=3 etcdctl snapshot status snapshotdb --endpoints=https://<IP>:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key
BASH
Perform new snapshot and verify
Perform new snapshot and verify

Taking a manual snapshot like this places a file in the working directory, the home directory in my case

Snapshot location
Snapshot location

This means when and if the snapshot is taken from inside the etcd pod it will save to the mounted volumes we found in the etcd.yaml file previously

So, taking a snapshot from outside let's you put that file in a secure location outside of the cluster itself. Make sure you also backup your certificate files that are needed for communicating with the cluster

Restore the etcd cluster
We can restore an etcd cluster from a snapshot taken on a cluster running the same MAJOR and MINOR version, meaning that there could be different patch versions.

A restore operation can be done from a snapshot file, or a data directory. Restoring from a snapshot is actually classed as a disaster recovery option and it initializes a completely new etcd cluster with new etcd member id's etc.

For the CKA exam I suspect that a full restore of an etcd cluster is not to be performed, but I'll go through the steps anyway. This document has some details that might come in handy if and when testing a restore

The key here is to restore to a different directory, and then update the configuration of the etcd pod. Again, this is specific to environments running stacked etcd (inside Kubernetes). If running an external etcd cluster the restore steps are different.

Before restoring I'll quickly spin up a new deployment. This is done after the snapshot was taken so in theory this shouldn't be part of the cluster when restoring

Create new deployment
Create new deployment

A note before continuing: I didn't stop the kube-apiservers before performing the steps which is specified in the Kubernetes documentation. That could answer a few of the issues I had going through the steps

So first I'll restore the snapshot to a new directory, note that the directory should NOT exists - it's created in the restore process. I'll fetch the cluster endpoint details from the current /etc/kubernetes/manifests/etcd.yaml file

sudo ETCDCTL_API=3 etcdctl snapshot restore snapshotdb --name <NODE-NAME> --initial-cluster <NODE-NAME1>=https://<IP1>:2380,<NODE-NAME2>=https://<IP2>:2380,<NODE-NAME3>=https://<IP3>:2380 --initial-advertise-peer-urls https://<NODE-IP>:2380 --data-dir <NEW-DIRECTORY>
BASH
Restore snapshot to new directory
Restore snapshot to new directory

Note that in my screenshot I'm specifying the /var/lib/etcd-from-snapshot2 directory. This is just for this particular screenshot, I restored to the etcd-from-snapshot directory, but forgot to get a screenshot..

Now we need to get the etcd pods to start using this new directory by updating the /etc/kubernetes/manifests/etcd.yaml configuration file. Update the hostPath path in the volumes section to reflect the new directory. If you've copied the certificates to a new directory this will also have to be updated.

The etcd pods should automatically be restarted when saving this file, which they did for two of my nodes. The last node didn't update and kept it's old config and this prevented also the two updated ones to start.

Two new pods created, one still running old config
Two new pods created, one still running old config

I tried to delete the pods, but then the one node got stuck in a terminating state. I also tried to restart all nodes without that fixing this, so in the end I force deleted the pod, and this fixed the two pods stuck in pending state

Killed the last pod
Killed the last pod

I did a new restart of the kubelet on all nodes, and after a while the pod on the last node got recreated and an etcd pod was running on all nodes again

All etcd pods recreated and running
All etcd pods recreated and running

We can do a describe on of the pods to see what host paths it's using

kubectl describe pod -n kube-system <POD-NAME>
BASH
Describe pod to see host path
Describe pod to see host path

Now, let's check out if I lost that newly created deployment

Deployment is gone
Deployment is gone

When pulling a new etcdctl endpoint status we can check the member id's which should change per the documentation

Etcdctl endpoint status
Etcdctl endpoint status

In my case one of the nodes did not change it's id. This is also the node that didn't terminate it's etcd pod, not sure if that's related, but it's something to investigate when time permits. For now I'm satisfied with having tested the steps.

Summary
This post has covered some basics around etcd in general and in conjunction with Kubernetes. It's meant to be a way for me to prepare for the CKA exam so it might be missing some steps needed in a "real-world" environment.

If you want more information about etcd please refer to their website. I've also found this article by Luc Juggery to explain things in a nice way

Thanks for reading!
```
