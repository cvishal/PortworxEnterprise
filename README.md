## Portworx Enterprise with IBM ROKS 
---

### Introduction:
This document helps user deploy portworx storage with IBM Cloud Red Hat Openshift Kubernetes Service (ROKS). It was created for the purpose of Watson Cloud Pak for AIOPs V 3.1 Deployment.

---
### Installation High Level Steps

- Provision ROKS Cluster 5 Worker Nodes (16Cores/64GB) - Classic Infra
- Use IBM Cloud Block Attacher Plugin, 
- Provision and attach Block Storage, using predefined script.
- Install Portworx Enterprise from IBM Cloud Catalog
- Validate you are able to create required storage classes.

-----

### Tools needed on developer laptop
- oc CLI
- ibmcloud CLI
- helm CLI

----

### Step 1  Install Attacher Plugin (block-attacher) using helm chart

- Make sure your have logged in to OCP Cluster using oc login command
- Run following commands

```
helm repo add iks-charts https://icr.io/helm/iks-charts
helm repo add ibm-charts https://raw.githubusercontent.com/IBM/charts/master/repo/stable
helm repo add ibm-community https://raw.githubusercontent.com/IBM/charts/master/repo/community
helm repo add entitled https://raw.githubusercontent.com/IBM/charts/master/repo/entitled
helm repo add ibm-helm https://raw.githubusercontent.com/IBM/charts/master/repo/ibm-helm

helm repo update

helm install block-attacher iks-charts/ibm-block-storage-attacher --namespace kube-system

```

- Make sure there is daemon set created and all pods are running

`oc get pod -n kube-system -o wide | grep attacher`

----

### Step 2 Setup Block Storage for Portworx on IBM Cloud  (Automated Via Script)

_Please note that if you don't want to use ibmcloud-block-storage-provisioner script, you can manually create new volume and attach to each instance of woker node. Please look at the instructions specified in Appendix A and skip this step 2 completly._


#### Please refer https://github.com/IBM/ibmcloud-storage-utilities

- Clone `git clone https://github.com/IBM/ibmcloud-storage-utilities.git`
- `cd ibmcloud-storage-utilities/block-storage-provisioner`

- Leverage `cvishal/ibmcloud-block-storage-provisioner:latest` and skip following 3 steps, but I encourage you to build your own docker image. Its quick.

```
1. Make sure you have python3 and docker on this machine
2. `make all`
3. Use your own docker image `ibmcloud-block-storage-provisioner:latest`
```

- Edit yamlgen.yaml
- Make sure it points to your ROKS cluster. Example below, cluster name is **waiops31dev**
- Give the volume size as per your requirements.

- Example:
```
cluster: waiops31dev  #  name of ROKS cluster
region:  us-south       #  cluster region
type: endurance          #  performance | endurance
offering: storage_as_a_service   #  storage_as_a_service | enterprise | performance
hourly_billing_flag: True
endurance:
  - tier:  4              #   [0.25|2|4|10]
size: [ 400 ]

```
_Please note: Above script will provision required block storage using your IBM Cloud account. Make sure you have proper permission to provision classic infrastructure_

- Collect following data points

- SL_USERNAME is a SoftLayer user name.  Example: 2xxxxxx_cvishal@in.ibm.com
- SL_API_KEY is a SoftLayer API Key. Login -> IAM -> API Keys -> View -> Classic Infra Key - Create one.

- Make sure your fresh login to ibmcloud `ibmcloud login --sso` before. 
- Run following command from same directory where yamlgen.yaml is present.
- If you have built your own docker image, please use it instead of using cvishal/ibmcloud-block-storage-provisioner

```
docker run --rm -v `pwd`:/data -v ~/.bluemix:/config -e SL_API_KEY=<classic_infra_key> -e SL_USERNAME=2XXXXXX_cvishal@in.ibm.com cvishal/ibmcloud-block-storage-provisioner
```
- Look newly generated script, example `pv-<clustername>.yaml`
- `oc apply -f pv-<clustername>.yaml`
- This will attach additional storage to ROKS Cluster.
- `oc describe pv | grep -i attachs` and wait untile you see status as attached.

### Step 3 Provision 'Portworx Enterprise' from IBM Cloud Catalog
- Select KVDB instead of etcd
- Give IBM Cloud API Key (Standard API Key from Access IAM ->  API Keys -> IBM Cloud API Key)
- Select your resource group -> OCP Cluster
- Click on Provision

### Step 4 validate 
- Go to command line and `oc get pods -n kube-system`
- Make sure all portworx related pods and up and running 
- Test the storage by creating PVC 

Example: 

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: testpvc
 namespace: default
spec:
 storageClassName: "portworx-shared-sc"
 accessModes:
 - ReadWriteMany
 resources:
   requests:
     storage: 1Gi
```
- Run this testpvc and make sure PV gets created and attached to this PVC.

------


## Portwork Cleanup Process (if needed)

_Please dont execute this if you have important system. This script will clean all the storage and portworx instance etc._

- When run into problem make sure you cleanup the storage
- Use Git project - https://github.com/IBM/ibmcloud-storage-utilities 
- cd px-utils/px_cleanup directory
- Run `px_cleanup.sh` which will clean Portworx Storage Classes, pods, daemonsets, etc and free Block Storage. It also clean Protworx Enterprise service etc
- After cleanup script, make sure you revoke access of storage from all worker nodes


----

## Some Tips and useful commands:

- Get attached stoarge for each node. Execute this after running the commad `oc apply -f pv-<clustername>.yaml`
- Get all nodes and for each node, run 'lsblk' command

Example: You have 3 nodes in your ROKS Cluster

```
oc get nodes
NAME          STATUS   ROLES           AGE   VERSION
10.185.2.39   Ready    master,worker   22h   v1.19.0+d856161
10.185.2.49   Ready    master,worker   22h   v1.19.0+d856161
10.185.2.51   Ready    master,worker   22h   v1.19.0+d856161


oc debug node/10.185.2.39 -- chroot /host lsblk
oc debug node/10.185.2.49 -- chroot /host lsblk
oc debug node/10.185.2.51 -- chroot /host lsblk

```
- Make sure you see the storage which you have requested to attach and not more or less.


- Create a small script which check Portworx Status on each node.

- vi check-pxctl-status.sh and add following lines (example for 3 node cluster)
```
PX_POD=$(kubectl get pods -l name=portworx -n kube-system -o jsonpath='{.items[0].metadata.name}')
kubectl exec $PX_POD -n kube-system -- /opt/pwx/bin/pxctl status
PX_POD=$(kubectl get pods -l name=portworx -n kube-system -o jsonpath='{.items[1].metadata.name}')
kubectl exec $PX_POD -n kube-system -- /opt/pwx/bin/pxctl status
PX_POD=$(kubectl get pods -l name=portworx -n kube-system -o jsonpath='{.items[2].metadata.name}')
kubectl exec $PX_POD -n kube-system -- /opt/pwx/bin/pxctl status

```
----

## Appendix A: If dont want to use script provided by IBM, you can manually create volume and attach to all worker nodes.

### Create the required Block Storage with IBM Cloud (400GB on each worker node)

- Get List of worker nodes </br>
`ibmcloud oc worker ls --cluster waiops31dev`

- Login to ibmcloud CLI </br>
`ibmcloud login --sso`

### For each worker node run following steps.
#### START HERE FOR EACH WORKER NODE
------
- 1. Provision 400GB Block Storage </br>
`ibmcloud sl block volume-order --storage-type endurance --size 400 --tier 4 --os-type LINUX --datacenter dal10 -b hourly`

- 2. Wait for volume to ready</br>
`ibmcloud sl block volume-list --order <order number>`

- 3. Get Details of storage (Target IP and LUN Id)</br>
`ibmcloud sl block volume-list` (Wait for volume to be ready)</br>
`ibmcloud sl block volume-detail <volume_ID>`</br>

- Example
```
vishal@~>ibmcloud sl block volume-detail 230521896
Name                       Value   
ID                         230521896   
User name                  DSW02SEL2137118-242   
Type                       endurance_block_storage   
Capacity (GB)              1500   
LUN Id                     1   
Endurance Tier             WRITEHEAVY_TIER   
Endurance Tier Per IOPS    4   
Datacenter                 dal10   
Target IP                  161.26.99.184   
# of Active Transactions   0   
Replicant Count            0   
```

- 4. Get list of all worker nodes and their private IPs </br>
- `oc get nodes -o wide` </br>
- Pick one node to attache storage. Node down the private IP of that Node </br>

- 5. Authorise newly provisioned block storage to worker node private IP </br/>
- `ibmcloud sl block access-authorize <volume_ID> -p <private_worker_IP>` </br/>

- 6. Note down host-iqn, username and password </br/>
- `ibmcloud sl block access-list 230521896` </br/>

Example:
```
vishal@~>ibmcloud sl block access-list 230521896
id          name            type   private_ip_address   source_subnet   host_iqn                                    username                    password           allowed_host_id   
161865392   10.94.118.199   IP     10.94.118.199        -               iqn.2021-04.com.ibm:dsw02su2137118-i161865392   DSW02SU2137118-I161865392   kafZp8dD5NB2mq2u   2326788   
```

- Create a pv.yaml for attaching storage </br/>
- Example</br/>

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: aiops31pv
  annotations:
    ibm.io/iqn: "iqn.2021-04.com.ibm:dsw02su2137118-i161865392"
    ibm.io/username: "DSW02SU2137118-I161865392"
    ibm.io/password: "vZP6Tb9EBqNbXNYv"
    ibm.io/targetip: "161.26.99.184"
    ibm.io/lunid: "1"
    ibm.io/nodeip: "10.94.118.199"
    ibm.io/volID: "230521896"
spec:
  capacity:
    storage: "400Gi"
  accessModes:
    - ReadWriteOnce
  hostPath:
      path: /
  storageClassName: ibmc-block-attacher

```

- Run  `oc apply -f pv.yaml`
- Run `oc decribe pv <pvname>` and make sure state is attached.

#### LOOP ENDS HERE FOR EACH WORKER NODE

------
- Once all Block Storage is provisioned and PVs are created, you can proceed for Portworx Enterprise Service installation.