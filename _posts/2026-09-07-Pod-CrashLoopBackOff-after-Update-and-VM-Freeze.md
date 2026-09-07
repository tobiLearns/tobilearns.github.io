---
title: Pod CrashLoopBackOff after Update and VM Freeze
date: 2026-09-07
categories: [Homelab, Kubernetes]
tags: [kubernetes, k3s, containerd, vm]
---
Recently I had some trouble in my cluster with a crashing Pod after an update from my renovate bot.  
I run a k3s cluster on a VM under Proxmox. The cluster is managed by Flux. Version updates are done in a semi-automatic way by a renovate bot, which creates pull requests in my repo. After reviewing these pull requests, I merge them manually. Usually, this goes smoothly without any problems.  
But recently, after Flux reconciled with the latest pull requests I had merged, it happened that suddenly this k3s-VM as well as my devops-VM (which I use to manage my cluster), were frozen.

### Troubleshooting
I restarted the devops-VM and recognized, that it ran out of disk space. This wasn't caused by the VM's own declared disk space, but by hitting the disk space limit of the underlying Proxmox node (which happened due to some storage-over-commitment on the Proxmox host - but that's a different story).

After freeing some disk space I also restarted the k3s-VM. Flux reconciled again with my git repo and the cluster was running again. But I saw, one Pod was crashed: The flux-system/helm-controller was stuck in a CrashLoopBackOff status. It was one of the Pods for which a version update was done with the latest pull requests from renovate.

So I took a look into the logs of this Pod:
````bash
# kubectl logs -n flux-system helm-controller-<id> --previous
exec /usr/local/bin/helm-controller: exec format error
````
Search results for this error wanted to convince me, that this was caused by an image, built for a different CPU architecture. This seemed rather unlikely, since I neither changed the physical machine, my cluster was running on, nor the type of image. It was a new version.

I thought, something was broken during the interrupted reconciliation of Flux. And since Flux couldn't fix it itself, my first idea was to revert this failed last pull request (Flux successfully reconciled the cluster into the previous state and all Pods were running fine) and then apply it again (Flux reconciled again to the new version of the helm-controller and the Pod crashed again).

Then I saw in the events of the Pod, the message "Container image ghcr.io/fluxcd/helm-controller:v1.5.3 already present on machine". So, if there was really something wrong with the downloaded image, it seemed that this same broken image was used again.
I found out how to remove the present image:
````bash
sudo k3s crictl rmi ghcr.io/fluxcd/helm-controller:v1.5.3
````
and deleted the crashed Pod.
When the Pod was created again, I saw in the events, that the image was pulled again and eventually the Pod was up and running fine.

After all this trouble I looked again into this "exec format error". And I found out, that a corrupted/truncated image is indeed the second well known reason for this error ;-)

### Little Dive into containerd's Internals:
When pulling an image, containerd performs two main steps to store this image in a usable format to create a container \[1\]:
1. Store the image content in the *content store* and
2. *Unpack the image*: Create a *committed snapshot* for each layer of the image.

In this first step containerd stores all parts of that image (index, manifests, config and compressed layer blobs) in its *content store* ``/var/lib/containerd/io.containerd.content.v1.content/blobs/`` (or ``/var/lib/rancher/k3s/agent/containerd/...`` for k3s). At last, after the data was flushed onto the disk and a successful integrity check was done, a rename of the written file to its actual name is done. That way it can be assured, that an interruption of that step doesn't end with an incomplete file on disk \[2\].

In the second step, containerd unpacks these image layers into a filesystem (known as *snapshot*).
These snapshots are stored under ``/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots`` (or the corresponding k3s directory).
The directories for the snapshot are prepared \[3: L303 ``func unpack``, 4: L265 ``func Prepare``, 4: L428 ``func createSnapshot``\] and a digest of the processed stream of the decompressed layer content (the later snapshot) is calculated \[5: L62 ``func Apply``\] and compared to the expected diffID. If the check succeeds, the snapshot is committed \[4: L297 ``func Commit``\], i.e., this snapshot is marked as committed in the metadata.
Now, actually this also prevents from storing corrupted snapshots (images). But it can happen, that the data, written in the ``Apply`` call, still reside in memory only and aren't flushed onto the disk at the time, when ``Commit`` is called.
If then, after the commit, the system crashes, the snapshot is lost (or incomplete), while the corresponding metadata says, it is complete. When the system is rebooted, containerd sees a complete pulled image in the metadata, but on disk only the corrupted/incomplete files exist.

There exists an option *image_pull_with_sync_fs* in containerd (and k3s) to also perform a flush of the written snapshot before it is marked as committed. By default this option isn't activated. It can cause a massive sync operation, which would impact the performance of other running processes on the system.

### References
- \[1\] [Content Flow](https://github.com/containerd/containerd/blob/main/docs/content-flow.md)
- \[2\] [plugins/content/local/writer.go](https://github.com/containerd/containerd/blob/v2.3.4/plugins/content/local/writer.go)
- \[3\] [core/unpack/unpacker.go](https://github.com/containerd/containerd/blob/v2.3.4/core/unpack/unpacker.go)
- \[4\] [plugins/snapshots/overlay/overlay.go](https://github.com/containerd/containerd/blob/v2.3.4/plugins/snapshots/overlay/overlay.go)
- \[5\] [core/diff/apply/apply.go](https://github.com/containerd/containerd/blob/v2.3.4/core/diff/apply/apply.go)
