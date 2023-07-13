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
5. plug in the vcenter details and connect
   1. when it prompts select the option to deploy mgmt cluster
   2. add a ssh key
6. becuase we are in a small environment choose a development cluster
7. select NSX ALB as the loadbalancer type
   1. leave control plane endpoint blank
8. fill in the NSX ALB details
   1. you can copy the cert from the AVI UI under templates-> security ->SSL/TLS Certificates
   2. select default cloud
   3. select the correcrt network settings
9. skip through metadata
10. select the correct locations for the resources in the cluster

----**to simulate an issue with netwokring**----

11. select the network for k8s network settings in our case let's select an incorrect network to simulate the failure. this is where nodes get deployed

----**to deploy sucessfully**----

11. select the network for k8s network settings, this is where nodes get deployed.

----
12. continue through to review configuration
13. export configuration and then deploy. 
14. copy the CLI command for when it fails


#### Troubleshoot the issue

1. get the kubeconfig for the kind cluster and start looking around.
   
```bash
kind get clusters

kind export kubeconfig --name <cluster-name> --kubeconfig ./kind-config

export KUBECONFIG=./kind-config
```
2. check for any failing pods

```bash
kubectl get pods -A 
```

3. check cluster api objects for errors

```bash
kubectl get cluster -n tkg-system -o yaml
kubectl get kcp -n tkg-system -o yaml
kubectl get md -n tkg-system -o yaml
kubectl get machines -n tkg-system
kubectl get vspheremachines -n tkg-system
```

4. check avi UI to see if there are any errors
 
5. check capi logs

```bash
kubectl get pods -A | grep cap

kubectl logs -f -n capv-system <capv pod> 
```

6. check to see if everything is getting IPs(VMs, LB etc.)
   1. this can be checked in the AVI UI and vcenter UI, make sure that the k8s nodes are getting ips




## Integrate mgmt cluster with TMC

### Login to TMC with the tanzu cli

1. install the tmc plugins into the tanzu cli

```bash
#enter the api token on prompt
tanzu context create --endpoint <your tmc endpoint> --name tmc
```
2. once the context is created it will download all of the plugins needed for tmc

### Add the management cluster to TMC

1. login to the mgmt cluster kube context
```bash
tanzu mc kubeconfig  get --admin
kubectl config use-context <admin context>
``` 
2. add the management cluster through the TMC CLI. We can also do this through the UI.
   1. replace the cluster name and cluster group in the below command

```bash
cat <<EOF > mgmt.yaml
fullName:
  name: <cluster-name>
spec:
  defaultClusterGroup: <cluster-group>
  kubernetesProviderType: VMWARE_TANZU_KUBERNETES_GRID
type:
  kind: ManagementCluster
  package: vmware.tanzu.manage.v1alpha1.managementcluster
  version: v1alpha1
EOF

#register the new mgmt cluster with tmc
tanzu tmc management-cluster create -f mgmt.yaml
tanzu management-cluster register <cluster-name> -k ~/.kube/config
```

2. check the agent status in the management cluster with kubectl

```bash
kubectl get pods -n vmware-system-tmc
```

3. check in the TMC UI and see if everything is healthy with the mgmt cluster

## Deploy clusters


### Using Tanzu CLI

[Official docs](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.2/using-tkg-22/workload-clusters-configure-vsphere.html)

1. Copy the configuration templaet from the docs above. additionally the one used to create the management cluster as a good starting point for copying some of the vsphere variables. The one used to create the managemnent cluster will be in `~/.config/tanzu/tkg/clusterconfigs/`. all of the variables that are available are listed [here](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.2/tkg-deploy-mc-22/mgmt-deploy-config-ref.html)

2. Edit the cluster config and update any setting that should be changed. 

3. with tkg 2.x cluster class based clusters were introduced. we can still use the TKG legacy yaml config which is what was created above. there are two options for using it now. we can either have it convert the config to a cluster class config and save it for review or auto apply. in this case we will have it auto apply. you can still use `--dry-run` to get the output without applying

```bash
tanzu config set features.cluster.auto-apply-generated-clusterclass-based-configuration true
```

1. Create the cluster. 

```bash
tanzu cluster create -f workload-cluster.yaml
```


### Using TMC

this can be done using the CLI or the UI. In this example we will create through the UI and then run the commands from the CLI to get the cluster cofniguration and create a new cluster using the cli.

**make sure clusters have enough cpu to run TMC agents. 1 node with 2vcpu is too small**

#### Create using the UI

follow the prompts in the UI to create a cluster. docs [here](https://docs.vmware.com/en/VMware-Tanzu-Mission-Control/services/tanzumc-using/GUID-C778E447-DDBB-49FC-B0B2-A8012AC56B0E.html#GUID-C778E447-DDBB-49FC-B0B2-A8012AC56B0E)


#### Create using the CLI

Using the previsouly created cluster we will generate a yanml config to create a cluster through the cli.

1. get the existing cluster config

```bash
tanzu tmc cluster get <cluster-name>  -m <mgmt cluster name> -p <provisoner name> > new-tmc-cluster.yaml
```

2. cleanup the yaml and rename cluster
   1. remove the entire `meta` section
   2. change the cluster name
   3. remove the entire `status` section
   4. remove the variable for `apiServerEndpoint`
   5. save the file

3. create the cluster using the new file

```bash
tanzu tmc cluster create -f new-tmc-cluster.yaml
```
### Using CAPI Cluster CRD

This was referenced above when talking about cluster class based clusters. TKG now allows for use of the `cluster` CRD along with cluster classes to deploy TKG clusters. this means you can now manage clusters with native k8s yaml.

legacy to cluster class variable reference -  [docs](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.2/using-tkg-22/workload-clusters-legacy-cc.html)


1. using the same configuartion file from the previous step we are going to generate a cluster object spec to start with.

```bash
tanzu cluster create -f workload-cluster.yaml --dry-run > workload-cc.yaml
```
2. edit this file and change the cluster name and modify any field that you want. Inspect how this object based cluster's field relate to the legacy variables using the referenced doc.

3. create the new cluster

```bash
kubectl apply -f workload-cc.yaml
```

4. monitor the cluster status with kubectl or tanzu cli

```bash
#check the conditions on the status
kubectl get cluster <clustername> -o yaml

tanzu cluster get <clustername>
```

## Use a custom ADC when creating a cluster

### create an ADC


1. create an ADC to enable NPL. take the existing ADC that is default and copy it to a file
```bash
kubectl get adc install-ako-for-all -o yaml > npl-adc.yml
```
2. modify the ADC to enable NPL and add labels to make the new ADC selectable. this is outline in the [Docs](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.2/tkg-deploy-mc-22/mgmt-reqs-network-nsx-alb-ingress.html#nodeportlocal-for-antrea-cni-2). First we need to do some cleanup of the file
   1. remove everything except for the `name` from the `metadata` section
   2. change the name to `npl-adc`
   3. add  `spec.clusterSelector.matchLabels. nsx-alb-node-port-local-l7: "true"` 
   4. add `spec.extraConfigs.cniPlugin: antrea`
   5. add `spec.extraConfigs.ingress.serviceType: NodePortLocal`
   6. change `spec.extraConfigs.ingress.disableIngressClass: false`
   
3. apply the new ADC into the mgmt cluster
 ```bash
 kubectl apply -f npl-adc.yml
 ```  
   

###  Deploy cluster and reference ADC with TMC

using our previous config file we have for creating a cluster with the TMC CLI we can add the new label to associate the newly created ADC with that cluster. 

1. using the label from above `nsx-alb-node-port-local-l7":"true"` add this to the cluster config when deploying. The config file should have the below added. 

```yaml
labels:
    nsx-alb-node-port-local-l7: "true"
```

2. validate it's using the new adc. the tkg cluster labels should now have a label for the avi ADC 

```bash
kubectl get clusters <cluster-name> -o yaml | grep -i avi
```
   

### Install a Tanzu package

Tanuz packages can be install with the tanzu cli, tmc, or through gitops. These are supported carvel packages for common OSS software. 

## Install Fluent bit using TMC

1. go into the tmc UI and go to the catalog and select fluent bit. this will show us all availabel values etc. This is helpful for seeing what's possible. 
2. fluent bit config is exposed through the package values, anything in the `config` section is direct from the fluent bit [docs](https://docs.fluentbit.io/manual/)
3. create a cluent bit config that matches your needs and add it to the `configs/fluent-install.yaml` file under the `spec.inlineValues.fluent_bit.config` section. Also replace the values for the cluster naming.
4. install the package using the tanzu cli 

```bash
tanzu tmc tanzupackage install create -f configs/fluent-install.yaml
```

## Inspect the objects that it creates

everything is a k8s object so it can be managed wit gitops also.

1. check the package repository

```bash
kubectl get pkgr -A
```

2. check the package install

```bash
kubectl get pkgi -A
kubectl get pkgi -n <namespace> <package name>
```

3. check the app

```bash
kubectl get apps -A
kubectl get apps -n <namespace> <app name>
```

4. check the values

```bash
kubectl get pkgi -A
kubectl get pkgi -n <namespace> <package name>
#check the name of the secret that is used for the values under spec.values.secretRef
kubectl get secrets -n  <namespace> <secret name> -o yaml
```

# Day 2 Ops

## Modify a cluster

Modifying a cluster can be done in a number of ways. We will cover modifying using CAPI as well as TMC.

### using the CAPI cluster CRD

let's modify the number of cpus that a worker node has.

1. open up the cluster yaml that we previosuly used to create the cluster. Look for the section that shows how many cpus a worker has. it looks like the below yaml

```yaml
- name: worker
      value:
        count: 1
        machine:
          diskGiB: 40
          memoryMiB: 8192
          numCPUs: 2
```
2. update the `numCPUs` to 6
3. apply the modified cluster yaml into the mgmt cluster.

```bash
kubectl apply -f workload-cc.yaml
```
4.  validate a new node is provisoned and the old one deleted.

```bash
kubectl get machines -n default | grep -i <clustername>
```
### Using TMC

let's modify the number of worker nodes a cluster has. this can be done using the UI as well.

1. edit the cluster yaml that we used previosuly for creating the cluster with tmc and change the worker node count. the field to change is `spec.topology.nodePools.spec.replicas`
2. update the cluster

```bash
tanzu tmc cluster update -f new-tmc-cluster.yaml
```

3. check the TMC GUI and validate the cluster has added a node
4. check the tanzu cli and validate it shows a new node

```bash
tanzu cluster get <cluster-name>
```

## Rotate the avi cert

### Simulate failure and fix

1. change the cert on AVI controller
   1. [docs](https://avinetworks.com/docs/latest/how-to-change-the-default-portal-certificate/)
   2. create a new cert and remove the old one. this will make the cert on tkg become out of date. simulating an expirng etc.
2. show what this looks like when it's broken in AKO. you should see X509 errors. 

```bash
#delete the pod 
kubectl delete pod ako-0 -n avi-system
kubectl logs -f -n avi-system ako-0
```

3. rotate the cert and show fixed AKO
   1. [docs](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.2/tkg-deploy-mc-22/mgmt-reqs-network-nsx-alb-manage.html)
   2. get the certificate from avi and base64 encode it
   3. patch the secret in the mgmt cluster

```bash
kubectl patch secret/avi-controller-ca  -n tkg-system-networking -p '{"data": {"certificateAuthorityData": "<ca-data>"}}'
```
   4. restart the ako operator pod in the tkg-system-networking ns


## Troubleshoot Tanzu packages

using the same package install from above let's make it fail the install and troubleshoot it

### deploy failed package

1. modify the package install yaml from the previous step. We will change one of the values to be incorrect. in the fluent bit values spec change the `podAnnotations: {}` to `podAnnotations: []`
2. apply the change and check the package status. it should be in a reconcile failed state.

### troubleshooting

1. check the error message on the package install. run the below command and check the `usefulErrorMessage`

`kubectl get pkgi -n <namespace> <package name> -o yaml`

2. for even more details we can look at the app, the package is bubbling up app errors

```bash
kubectl get apps -n <namespace> <app name> -o yaml
```




## Troubleshoot failing cluster deployment



## Upgrade TKG workload cluster