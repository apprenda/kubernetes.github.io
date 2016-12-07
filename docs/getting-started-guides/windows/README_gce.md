# Running Windows Server Containers atop Kubernetes and Google Cloud Platform

## Prerequisites
In Kubernetes version 1.5, Windows Server Containers for Kubernetes is supported using the following:

1. A Google Cloud Platform project 
2. Kubernetes control plane running on existing Linux infrastructure (version 1.5 or later)
3. Kubenet network plugin (the default) setup on the Linux nodes
4. One or more nodes running Windows Server 2016 (official GCP image)

## Networking

Windows Containers networking is not as powerful and straightforward as is the case with Linux. Therefore, and until Windows Containers
supports features like network overlay and bridge masquerading, one is limited to L3 routing when achieving the networking scheme
Kubernetes needs to be in place, namely:

- Each worked node (GCE VM) is assigned one IP address from the GCE network in use, from now on referred to as _node network_.
- Each worker node is manually assigned a unique subnet to be allocated to pods, from now on referred to as _pod network_.
- A unique cluster service subnet is to be allocated for `kube-proxy` usage.

**Attention:** aforementioned subnets **must** not overlap.

## Configure worker nodes

All commands from this point on rely on variables that one must adapt according to one's GCE environment.
Below is a list of words one will need to replace in order for commands to work properly.

- `<PROJECT_ID>` - GCP project identifier, e.g. `my-gcp-project`.
- `<ZONE>` - GCP zone to use, e.g. `us-east1-d`. 
- `<NETWORK_NAME>` - GCP subnetwork to use, e.g. `default`. 
- `<POD_SUBNET>` - the pod subnet of your choosing, e.g. `192.168.0.0/16`. 
- `<POD_SUBNET_ALLOCATED_TO_NODE>` - the pod subnet allocated to a certain worker node, e.g. `192.168.1.0/24`.
- `<POD_SUBNET_ALLOCATED_TO_NODE_GATEWAY>` - the default gateway to a pod subnet allocated to a certain worker node, e.g. `192.168.1.1`.

### Bootstrap nodes

For the sake of simplicity, the cluster we're about to build has only two (2) worker nodes. And following are the instructions on how to bring them up.

**Node 1**

```shell
gcloud compute --project <PROJECT_ID> instances create "kubewin-node-1" \
    --zone <ZONE> \
    --machine-type "custom-2-2048" \
    --subnet <NETWORK_NAME> \
    --can-ip-forward \
    --maintenance-policy "MIGRATE" \
    --scopes default="https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring.write","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" \
    --image "/windows-cloud/windows-server-2016-dc-v20161115" \
    --boot-disk-size "200" \
    --boot-disk-type "pd-standard" \
    --boot-disk-device-name "kubewin-node-1"
```

**Node 2**

```shell
gcloud compute --project <PROJECT_ID> instances create "kubewin-node-2" \
    --zone <ZONE> \
    --machine-type "custom-2-2048" \
    --subnet <NETWORK_NAME> \
    --can-ip-forward \
    --maintenance-policy "MIGRATE" \
    --scopes default="https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring.write","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" \
    --image "/windows-cloud/windows-server-2016-dc-v20161115" \
    --boot-disk-size "200" \
    --boot-disk-type "pd-standard" \
    --boot-disk-device-name "kubewin-node-2"
```

**Notes**

- The Windows Server 2016 image used is the latest, as of the time of this writing, provided by GCP. If you're using an older or tailored image, proceed at your own risk.
- The custom machine type selected here is for testing purposes only. Feel free to choose the machine type that best fits your needs.
- IP Forwarding **must** be enabled.
- Disk size has been set to 200GB because [I/O performance is affected with smaller disks](https://developers.google.com/compute/docs/disks#pdperformance).

### Enable Windows Containers

After worker nodes are brought up, one must set credentials in order to be able to log-in.

**Node 1**

```shell
gcloud beta compute --project <PROJECT_ID> reset-windows-password "kubewin-node-1" --zone <ZONE>
```

**Node 2**

```shell
gcloud beta compute --project <PROJECT_ID> reset-windows-password "kubewin-node-2" --zone <ZONE>
```

**Notes**

- The step described above can safely run multiple times, e.g. in case one loses an instance password.

One should be able to access the nodes through RDP. More information on how to achieve this is detailed [here](https://cloud.google.com/compute/docs/instances/windows/connecting-to-windows-instance).

Now, log-in into each of the nodes and [enable Windows Containers](https://cloud.google.com/compute/docs/containers/#docker_on_windows).
A reboot will be required.

### Set-up worker nodes networking

By default, Docker is not ready for running Kubernetes pod containers. The following steps are needed.

### Set-up container networking

#### Add Microsoft Loopback NIC adapter

One needs to add a new NIC to hold the pod network. In order to do so, execute *step 6*, and this step alone,
from [these instructions](https://cloud.google.com/compute/docs/instances/windows/configuring-static-internal-ip-windows).

#### Override Docker networking 

By default, when Docker daemon starts it will create a NAT network, if it doesn't exist already. In order to prevent that,
one needs to create a file, `C:\ProgramData\docker\config\daemon.json`, and add the following to it:

```json
{
    "bridge" : "none"
}
```

Save the file.

Now, let's reset the container networking setup Docker created before. 

```shell
Stop-Service docker
Get-ContainerNetwork | Remove-ContainerNetwork
Get-NetNat | Remove-NetNat
New-ContainerNetwork -Name podnet -Mode Transparent -NetworkAdapterName "Ethernet 2" -SubnetPrefix <POD_SUBNET_ALLOCATED_TO_NODE> -GatewayAddress <POD_SUBNET_ALLOCATED_TO_NODE_GATEWAY> 
Start-Service docker
netsh int ipv4 set address "vEthernet (HNSTransparent)" static <POD_SUBNET_ALLOCATED_TO_NODE_GATEWAY> 255.255.255.0
```

#### Enable IP Forwarding

```shell
netsh interface ipv4 set interface "Ethernet" forwarding=enabled
netsh interface ipv4 set interface "vEthernet (HNSTransparent)" forwarding=enabled
```

#### (Optional) Enable Docker daemon over TCP

In order to connect to the Docker daemon over TCP, the service must be reconfigured to listen on TCP.

```shell
dockerd --unregister-service
dockerd -H npipe:// -H 0.0.0.0:2375 --register-service
```

## Configure GCP Networking

### Routes

GCP Routes are needed for inter-node container communications. One route shall be created per worker node pod subnet so that all nodes know how to reach any pod in the cluster.

**Node 1**

```shell
gcloud compute --project <PROJECT_ID> routes create "kubewin-node-1" \
    --destination-range <POD_SUBNET_ALLOCATED_TO_NODE> \
    --network <NETWORK_NAME> \
    --next-hop-instance "kubewin-node-1" \
    --next-hop-instance-zone <ZONE> \
    --priority "1"
```

**Node 2**

```shell
gcloud compute --project <PROJECT_ID> routes create "kubewin-node-2" \
    --destination-range <POD_SUBNET_ALLOCATED_TO_NODE> \
    --network <NETWORK_NAME> \
    --next-hop-instance "kubewin-node-2" \
    --next-hop-instance-zone <ZONE> \
    --priority "1"
```

### Firewall

With routes in place, the last step in networking configuration is to create the needed firewall rules. 

```shell
gcloud compute --project <PROJECT_ID> firewall-rules create "allow-kubewin-podnet" \
    --allow tcp,udp \
    --network <NETWORK_NAME> \
    --source-ranges <POD_SUBNET> \
    --description "Allow Kubernetes pod network inter-node communication."
```


## Future work

- Automate pod subnet assignment to nodes - leverage on the following `controller manager` flags, i.e. `--cluster-cidr=<POD_SUBNET>` and `--cloud-provider=gce`.