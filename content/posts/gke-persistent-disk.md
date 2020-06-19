---
title: Preventing your PVC from being deleted in GKE
---

# The story

This one totally caught off guard. I have this Kubernetes cluster I managed, and well, one day I was doing my thing. And I noticed that my laptop was low on resources, so I decided to clean things up. Cleared out a few namespaces and everything was good to go. Well, not 3 seconds later, someone pings on a chat channel that their dev environment is gone. Well...crap. I double check my `kubectl` context, and yup, sure enough. I was not connected to my local instance, I was connected to the dev Kubernetes cluster. Oops!

Thankfully this was only dev, and not production or anywhere else important. But it still stood I blew up the cluster, and needed to rebuild it, and quickly. This post isn't how I brought that cluster back online (spoiler: we have tools and such to do that quickly), but the few lessons that came out of that.

# What are PVCs

Let's make sure we are on the same page. If you don't know. Persistent volume claims (PVC) are a way for you to get permentant disk storage in your Kubernetes cluster. This means if your pod dies, or is deleted, you can have your data persist. This is all fine and dandy, but by default. Those PVCs are a namespaced resource. Which means, if you delete the namespace. That resource is also deleted.

# Using persistent disks in GKE

To prevent your data from getting lost in the event that the namespace is deleted. You need to attach that data to a disk that lives outside of Kubernetes control. To do this, we will use a disk created in Google cloud, and attach to that disk inside our Kubernetes environment. This is [documented in GKE]() but let me give you the easy steps of getting there:

First, let's create the disk:

```
gcloud compute disks create my-disk
```

Next, let's attach to that disk in a pod:

```

```


# Migrate to use that disk

So, creating a disk is rather straight forward, but what if you already have a disk and want to migrate over to this? We can actually do that. All PVCs in GKE do actually have a disk attached to them. They are just managed outside of the cluster itself. So here is the general workflow you will need to go through to migrate over to something more permanent:

1. - Scale down your pods
1. - Take a snapshot of your current disk
1. - Delete your current deployment
1. - Create a disk from the snapshot
1. - Deploy again with the PVC attacked to the disk

# Step: Scaling down the pods

To ensure that all the data is done being written to the disks, and nothing changes the data mid flight. Its a good idea to make sure all pods that are using the disks are scaled down to 0. If you are using a deployment, this is as easy as `kubectl scale --replicas=0 deployment/my-stuff`. Then verify that all the pods are gone before proceeding on to the next step.

# Step: Snapshotting your current disk

To get us to a new disk, we need to go through and create a snapshot of the data you currently have, to seed the permanent disk. Whats need is that we can actually script this next part out. Here is what this script would look like:

```
# Capture properties of disk
VOLUME_NAME=$(kubectl get pvc my-pvc -o=jsonpath='{.spec.volumeName}')
DISK_NAME=$(gcloud compute disks list --filter="name~''$VOLUME_NAME''" --format="value(name)")
DISK_ZONE=$(gcloud compute disks list --filter="name~''$VOLUME_NAME''" --format="value(zone)")

gcloud compute disk snapshot --zone $DISK_ZONE --snapshot-names my-snapshot $DISK_NAME
```

First we had to capture data from the PVC. In this example `mv-pvc` would be the name of the PVC you create in your cluster. We capture the volume name which is the name of the disk created outside of the cluster.

Next, we need to capture some information about the disk itself. We need to know the full disk name, along with which zone it was created in.

From there, we then have all the information we need to create our snapshot `my-snapshot` (all these names can be tweaked, and are given as examples).

# Step: Deleting your deployment

Next, let's go ahead and delete your deployment. If you want a simple one liner, you can use `kubectl` to delete everything: `kubectl delete all,pvc,pv --all-namespaces -l app=my-app`. The label (`-l`) assumes you have some kind of annotation attached to your resources.

# Step: Creating the disk

Now that we have everything cleaned up, let's create the new disk.

```
gcloud compute disks create my-disk --size=1G --source-snapshot=my-snapshot --type=pd-ssd
```

Fairly straight forward. This will create the disk with a 1G limit. We use the SSD disk type because we are under 100GB for this example. But tweak it for your needs.

# Step: Deploying to attached disk

Now that we have the disk ready, we need to redeploy our service back into Kubernetes. But this time, we need to ensure that your PVC attaches to the disk. We will use a manifest like this:

```
```

If all goes well, all your data is still present and you have no migrated over to a more permanent disk.
