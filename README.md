# fcos-k8s-vmware
Install and setup of HA Kubernetes on VMWare using Fedora CoreOS

## Getting Started

* Download Fedora CoreOS OVA for VMWare. 

  This was tested using: https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/33.20210117.3.2/x86_64/fedora-coreos-33.20210117.3.2-vmware.x86_64.ova

* To generate the Ignition Configuration file, you need either `docker` or `podman` installed (if using podman, edit config.sh to change docker to podman).
  
* Edit the file `kubernetes.fcc` and add a public SSH key that will be used for login to the nodes.

* Run config.sh and pipe to pbcopy (on macOS) to copy the resulting base64 encoded string to the clipboard.
  ```
  config.sh | pbcopy
  ```
  
* Deploy the CoreOS OVA to VMWare, when asked for `Ignition config data` paste the base64 string from the last step. For `Ignition config data encoding`
  enter `base64`
  
* After the OVA is deployed, you can get the IP from the console and SSH via `core@<IP ADDRESS>`

## Installing Kubernetes

The first node you create should be a control plane node, ex. `k8s-controlplane-1`.

Login to the node as `core` and run `k8s-install.sh` to configure the machine for Kubernetes (as well as installing open-vm-tools and other supporting packages). Follow the instructions and reboot once the install completes.

After reboot, run `k8s-init.sh` to configure the first control-plane node. When asked for the `Virtual IP` assign a static IP on the same network as the VM *but not in the DHCP range*. This will be the kube-vip virtual IP address used to access the Kubernetes API.

When this step completes, the initial control plane should be ready. Take note of the `token`, `certificate`, and `CA Cert Hash` as part of the output as it will be needed later to join.

## Joining a Worker Node

Deploy the OVA again using the same `Ignition config data encoding` as before and run `k8s-install.sh` as before.

Reboot the node when instructed and then SSH as `core` when it is back up and running.

Run `k8s-worker-join.sh` and follow the instructions entering the token and discovery CA cert hash as prompted.

On the initial control-plane node, you can monitor the pods with `watch kubectl get pods -A` until all are ready.

Repeat this process to add additional worker nodes as needed.

## Adding additional control plane nodes for HA

Deploy the OVA again using the same `Ignition config data encoding` as before and run `k8s-install.sh` as before.

Reboot the node when instructed and then SSH as `core` when it is back up and running.

Run `k8s-controlplane-join.sh` and follow the instructions entering the token and certificate as prompted.

On the initial control-plane node, you can monitor the pods with `watch kubectl get pods -A` until all are ready.

Repeat this process again to add additional control plane nodes. Note that you should always have an odd number (such as 3) to ensure proper failover.

## Notes
You only have 2 hours to use the initial certificate key before it expires. 

Once it expires, you'll need to run:
```
sudo kubeadm init phase upload-certs --upload-certs
```
On the primary controlplane and copy the new certificate key for joining additional CP nodes.
