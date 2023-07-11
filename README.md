# TKG Operator Workshop


This workshop is run in a hosted lab and is not intended to be hands on by attendees. This doc outlines the different scenarios that instructors will run through along with some of the commands that can be run. 

# Architecture

**See Miro**

## Install

## Pre-reqs

* setup TKG jumpbox -  we will be using the service installer OVA from [here](https://marketplace.cloud.vmware.com/services/details/service-installer-for-vmware-tanzu-1?slug=true). this has the tanzu cli already installed, we won't actually use the service installer tools.
  * deploy this into your vsphere environment and connect to it via SSH. this will be our main terminal for doing the rest of the workshop
* deploy TKG TKR OVA - deploy the latest photon or ubuntu ova from [here](https://marketplace.cloud.vmware.com/services/details/tanzu-kubernetes-grid-1-1-1211?slug=true) and conver it to a template
* follow any other pre-reqs from the [TKG docs](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.2/tkg-deploy-mc-22/mgmt-reqs-prep-vsphere.html)

# Day 1 Ops 

## Deploy AVI

For this deployment we will be going minimal with only one network.

### Configure the default cloud
1. download and deploy the AVI controller OVA. [Official docs](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.2/tkg-deploy-mc-22/mgmt-reqs-network-nsx-alb-install.html)
2. set a static ip for the controller when deploying **Don't use first IP in range**
3. login to UI and setup admin account 
4. walk through the setup wizard
   1. no smtp
   2. default settings for multitenant
5. setup the default cloud under the infrastructure-> clouds tab
   1. change the orchestrator to vsphere
   2. leave IP address management and VS placement default
   3. set vcenter credentials
   4. uncheck content library
   5. save and relaunch
   6. select your network for management and configure the static IP info and pool. This will be defaulted to be used for SEs and VIPs. since this is a small environment that is fine for our use. add enough IPs in the range for VIP usage and SE placement. Also this range should not be in DHCP
   7. create an IPAM profile called tkg-ipam, use avi vantage ipam as the default
      1. select default cloud
      2. add your network as a usable network
   8. create a dns profile called TKG DNS
      1. Add a domain that we can delegate to 
   9. leave the rest default

### Edit the default SE group

Edit the default SE group to reduce IP use in a small environment

1.  set max SE to 1
2.  set the number of VS per SE to 20
   
### Update the default cert

1. under adminisration->system systems hit edit
2. delete the two certs under SSL/TLS Certificate
3. click the three dots on the right and create a new cert(or upload your own)
   1. give the common name field the IP address of the controller. we are not doing an fqdn for this setup.
   2. fill in the generic info for the cert
   3. add a SAN with the IP address also
   4. reload the page so the new certs takes over
### Turn off cloud licensing

1. under administration-> licensing click the gear and change the license to enterpise instead of cloud enterprise.

## Deploy management cluster

We will deploy the management cluster twice for this workshop. The first time we will simulate a failure and troubleshoot it. We will then destroy it and deploy a new cluster that is fully functional.

1. ssh to the jumpbox, using vscode we can forward the port to our local workstation to get the UI. this makes it easy to deploy TKG with the UI from a remote machine. 
2. run the create command
```bash
   tanzu management-cluster create --ui
```
3. select vsphere from the UI.
4. jump to one of the sections below for the simulated failure or real deploy


### Simulate and troubleshoot failure

1. plug in the vcenter details and connect
   1. when it prompts select the option to deploy mgmt cluster
   2. add a ssh key
2. becuase we are in a small environment choose a development cluster
3. select NSX ALB as the loadbalancer type
   1. leave control plane endpoint blank
4. fill in the NSX ALB details, this is where we will simulate failure by putting in an incorrect network name.
   1. you can copy the cert from the AVI UI under templates-> security ->SSL/TLS Certificates
   2. select default cloud
   3. in the mgmt cluster control plane vip network name we are going to put in an incorrect name
5. skip through metadata
6. select the correct locations for the resources in the cluster
7. select the network for k8s network settings in our case this is the same network. this is where nodes get deployed
8. continue through to review configuration
9. export configuration and then deploy. 
10. copy the CLI command for when it fails


#### Troubleshoot the issue

1. get the kubeconfig for the kind cluster and start looking around.
   
```bash
kind get clusters

kind export kubeconfig --name <cluster-name> --kubeconfig ./kind-config

```
2. check for any failing pods

3. check cluster api objects for errors

4. check capi logs

5. check avi UI to see if there are any errors

6. check to see if everything is getting IPs(VMs, LB etc.)
   
7. check AKO logs


### Deploy HA mgmt cluster

## Integrate mgmt cluster with TMC

1. add cluster through the UI
2. check the agent status in the management cluster with kubectl

```bash
kubectl get pods -n vmware-system-tmc
```



## Deploy clusters

### Using Tanzu CLI

### Using TMC

### Using K8s


## Custom cluster class

https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.2/using-tkg-22/workload-clusters-cclass.html


## Use a custom ADC when creating a cluster

### create an ADC
[Docs](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.2/tkg-deploy-mc-22/mgmt-reqs-network-nsx-alb-ingress.html#nodeportlocal-for-antrea-cni-2)

1. create an ADC to enable NPL. take the existing ADC that is default and copy it to a file
```bash
```

2. modify the ADC to enable NPL and add labels to make the new ADC selectable
3. apply the new ADC into the mgmt cluster
4. 

### Deploy cluster and reference ADC with Tanzu CLI

1. using the label from above `nsx-alb-node-port-local-l7":"true"` add this to the cluster config when deploying.

```bash
AVI_LABELS: '{"nsx-alb-node-port-local-l7":"true"}'

ANTREA_NODEPORTLOCAL: "true"
```

###  Deploy cluster and reference ADC with TMC

1. using the label from above `nsx-alb-node-port-local-l7":"true"` add this to the cluster config when deploying.
2. in the tmc UI this is a label when creating the cluster

### Install a Tanzu package

## Install Fluent bit

# Day 2 Ops

## Modify a cluster

### Using Tanzu ClI

### Using TMC

### Using K8s

## Rotate the avi cert

1. change the cert on AVI controller
   1. [docs](https://avinetworks.com/docs/latest/how-to-change-the-default-portal-certificate/)
2. show what this looks like when it's broken in AKO

```bash
```

3. rotate the cert and show fixed AKO
   1. [docs](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.2/tkg-deploy-mc-22/mgmt-reqs-network-nsx-alb-manage.html)


## Troubleshoot Tanzu packages

## Troubleshoot failing cluster deployment

## Upgrade TKG workload cluster