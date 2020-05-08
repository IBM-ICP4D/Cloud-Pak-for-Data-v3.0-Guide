# Install Openshift 4.3 on vSphere vCenter 7.0.0.10100


This document shows step by step method to install Red Hat OpenShift Container Platform 4.3 (OCP) on VMware vSphere with static IPs addresses using the openshift installer in UPI mode and terraform. There are other variations of the install where DHCP assigned IP addresses can be used but that would be explored later.

The current assumption is that vSphere is already installed and is ready for installation of OCP. In case you would like to setup vSphere, please follow this link

# Install topology 

![alt text](https://github.com/IBM-ICP4D/Cloud-Pak-for-Data-v3.0-Guide/blob/master/Installation/vsphere/images/2020-04-22%2000_22_10-vmware-ocp-install.pptx%20-%20PowerPoint.png "arch")

# Harware and Capacity Requirements

A Datacenter in vCenter should be able to provision below machines - 

| Node (hostnames)         | vCPU   |  RAM   |  Disk |  Quantity 
| ------------- |-------------| ------------- |-------------|-------------|
| Bastion (lb) | 2 | 4 | 200 GB | 1  |
| Masters (control-plane-[0-2])  | 8 | 32 | 200 GB | 3 |
| Workers (compute-[0-1]) | 16 | 64 |200 GB | 2 | 
| Bootstrap (bootstrap) | 4 | 16 | 120 GB | 1 | 
   
Note that these specifications are bigger than Redhat's recommendation keeping in mind that the cluster would be used to install Cloud pak for data.

# Software Prerequisites

   VMware vSphere version 6.5 or 6.7U2 or later instance with approriate permissions and below information
	
    - vCenter URL
    - vCenter login (See list of permissions in the end of the document)
    - vCenter password
    - datacenter name
    - cluster name 
    - datastore name (With atleast 500GB available space)

  Openshift artifacts
  
    - OCP pull secret 
    - OCP openshift-installer
    - RHCOS OVA image
    - oc client
  
  Internet connectivity from all the nodes
   
# Node Roles 

 
| Node          | Description   |
| ------------- |-------------|
| Bastion  | The bastion host act as the driver of the installatin  where we execute our openshift-installer and terraform commands. The node in this article doubles up as haproxy (load balancer) and http server. But in theory, if an external load balancer was used or provisioned in separate machine, this machine can be discarded post install |
| Masters  | Master nodes for OCP Cluster      |
| Workers  | Worker nodes for OCP Cluster also called Compute nodes      |
| Bootstrap | For clusters with installer-provisioned infrastructure, you delegate the infrastructure bootstrapping and provisioning to the installation program instead of doing it yourself. The installation program creates all of the networking, machines, and operating systems that are required to support the cluster.  One installation finishes, this node and be discarded.    |
 
#  Hostname and DNS Requirements 

As earlier Openshift, OCP 4.3 also needs each host/node to be resolved by DNS service. You would need to make entries for name resolution (A records), certificate generation (PTR records), and service discovery (SRV records) but before we talk more about the records, few important things to keep in mind around hostnames 

Base Domain -   A base domain is must for this setup as all other hostnames and service become part of the base domain. In this setup, cpd.xyz is our base DN
Cluster Name -  The vmware cluster under which all the resources would be created. This needs to be available before the install and must be DRS enabled. 
Resource Pool - This is the pool which gets created automatically as part of the terraform based machine provisioning in vSphere vCenter. All machines created becomes part of this resource group and are referred to as <resourcegroup>.<basedomain> in FQDNs for the cluster. Any nodes created with in this cluster  would eventually have <node>.<clusterid>.<basedomain> as FQDN
				We’ll use ocp4-cluster as a resource pool name and hence one of the compute has FQDN as : compute-0.ocp4-cluster.cpd.xyz
			

For this install, we had access to windows server and hence ended up configuring the windows DNS Manager for the configurations but irrespective of which DNS server is used, following records needs to be added.


| Names        | IP   |  Comments   
| ------------- |-------------| ------------- |
| bootstrap.ocp4-cluster.cpd.xyz | 192.168.176.100 |  | 
| control-plane-0.ocp4-cluster.cpd.xyz | 192.168.176.101 |  | 
| control-plane-1.ocp4-cluster.cpd.xyz | 192.168.176.102 |  | 
| control-plane-2.ocp4-cluster.cpd.xyz | 192.168.176.103 |  | 
| compute-0.ocp4-cluster.cpd.xyz | 192.168.176.104 |  | 
| compute-1.ocp4-cluster.cpd.xyz | 192.168.176.105 |  | 
| lb.ocp4-cluster.cpd.xyz | 192.168.176.134 | basion/lb/httpd server | 
| etcd-0.ocp4-cluster.cpd.xyz | 192.168.176.101 | Points to Master 1 | 
| etcd-1.ocp4-cluster.cpd.xyz | 192.168.176.102 | Points to Master 2 | 
| etcd-2.ocp4-cluster.cpd.xyz | 192.168.176.103 | Points to Master 3 | 
| lb.ocp4-cluster.cpd.xyz | 192.168.176.134 | basion/lb/httpd server | 
| api.ocp4-cluster.cpd.xyz | 192.168.176.134 | Kube API, Points to lb | 
| api-int.ocp4-cluster.cpd.xyz | 192.168.176.134 | Kube API, Points to lb | 
| *.apps.ocp4-cluster.cpd.xyz | 192.168.176.134 | Points to lb | 
| *.apps.ocp4-cluster.cpd.xyz | 192.168.176.134 | Points to lb | 
| *.apps.ocp4-cluster.cpd.xyz | 192.168.176.134 | Points to lb | 
| _etcd-server-ssl._tcp.ocp4-cluster.cpd.xyz |  SRV record point to etcd-0.ocp4-cluster.cpd.xyz with  priority 0, weight 10 and port 2380 |
| _etcd-server-ssl._tcp.ocp4-cluster.cpd.xyz |  SRV record point to etcd-1.ocp4-cluster.cpd.xyz with  priority 0, weight 10 and port 2380 |
| _etcd-server-ssl._tcp.ocp4-cluster.cpd.xyz |  SRV record point to etcd-2.ocp4-cluster.cpd.xyz with  priority 0, weight 10 and port 2380 |

Once all the changes are completed, flush the dns cache

```
ipconfig /flushdns
ipconfig /registerdns

```

Please validate the changes from bastion by running a forward and reverse lookup to ensure that these are setup properly and you are able to reach the DNS with proper resolution

`nslookup control-plane-2.ocp4-cluster.cpd.xyz` should return 192.168.176.103
`nslookup 192.168.176.103` should return control-plane-2.ocp4-cluster.cpd.xyz

If unable to resolve, please have a look at the `/etc/resolv.conf` to ensure the right DNS server is being used.


# Preparing Basion Node (lb.ocp4-cluster.cpd.xyz)

httpd : The http server provides the ignition for bootstrap via http. 
ha_proxy: the loadbalancer used for external loadbalancer (lb) used to load balance worker and master services. If there is an external hardware loadbalancer or other machine hosting this service, this can be skipped.

# Basic apps 

yum install -y vim bind-utlils git screen 


# Terraform

Install Terraform which is needed to setup the fully automated way of provisioning vmware machines and kick starting installation. 

Link: https://www.terraform.io/downloads.html

```
$export PATH=$PATH:~/bin 
$ mkdir ~/bin
$ wget https://releases.hashicorp.com/terraform/0.11.14/terraform_0.11.14_linux_amd64.zip
$ unzip terraform_0.11.14_linux_amd64.zip -d ~/bin/
```

# HAProxy Setup


``` 
$ yum install haproxy -y

```

Edit /etc/haproxy/haproxy.cfg and replace the content of the file. Change the IP addresses if you have a different range of IPs selected.

```
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------
frontend  main *:5000
    acl url_static       path_beg       -i /static /images /javascript /stylesheets
    acl url_static       path_end       -i .jpg .gif .png .css .js

    use_backend static          if url_static
    default_backend             app

#---------------------------------------------------------------------
# static backend for serving up images, stylesheets and such
#---------------------------------------------------------------------
backend static
    balance     roundrobin
    server      static 127.0.0.1:4331 check

#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend app
    balance     roundrobin
    server  app1 127.0.0.1:5001 check
    server  app2 127.0.0.1:5002 check
    server  app3 127.0.0.1:5003 check
    server  app4 127.0.0.1:5004 check

frontend openshift-api-server
    bind *:6443
    default_backend openshift-api-server
    mode tcp
    option tcplog

backend openshift-api-server
    balance source
    mode tcp
    server bootstrap 192.168.176.100:6443 check
    server control-plane-0 192.168.176.101:6443 check
    server control-plane-1 192.168.176.102:6443 check
    server control-plane-2 192.168.176.103:6443 check

frontend machine-config-server
    bind *:22623
    default_backend machine-config-server
    mode tcp
    option tcplog

backend machine-config-server
    balance source
    mode tcp
    server bootstrap 192.168.176.100:22623 check
    server control-plane-0 192.168.176.101:22623 check
    server control-plane-1 192.168.176.102:22623 check
    server control-plane-2 192.168.176.103:22623 check

frontend ingress-http
    bind *:80
    default_backend ingress-http
    mode tcp
    option tcplog

backend ingress-http
    balance source
    mode tcp
    server compute-0 192.168.176.104:80 check
    server compute-1 192.168.176.105:80 check

frontend ingress-https
    bind *:443
    default_backend ingress-https
    mode tcp
    option tcplog

backend ingress-https
    balance source
    mode tcp
    server compute-0 192.168.176.104:443 check
    server compute-1 192.168.176.105:443 check
 
```

Once saved, restart HAProxy and check the status

``` 
systemctl restart haproxy
systemctl status haproxy

```

# SSH keys

Since our CoreOS based VMs will be configured automatically, we have to provide a ssh public key to be able to log on to the nodes via ssh as user core.

Generate a ssh private key which would be used to automatically log on to the nodes for configurations and install. The user used with this key is `core`

``` $ ssh-keygen ```

Do not add any passwrods and just go with the defaults. The key pair would be generated at 
```
~/.ssh/id_rsa.pub
~/.ssh/id_rsa
```
The public key would be used to provision these machines and its private counter would be used for logging into the systems if needed, so keep it safe



# HTTPD Server

httd server is needed to pass the ignition files to the bootstrap server. We would run this in default setting at port 80, Feel free to change it if there is a conflict.


```

yum install httpd -y
systemctl start httpd
systemctl status  httpd 
systemctl enable httpd

```

Validate it by running

```

wget http://lb.ocp4-cluster.cpd.xyz

```

# Openshift Parts


Login to https://cloud.redhat.com/openshift/install/vsphere/user-provisioned with your credentials and and download all thats available there :)

    OpenShift installer [wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-install-linux.tar.gz]
	Pull secret for downloading the images off redhat [todo command]
	Red Hat Enterprise Linux CoreOS (RHCOS) [wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/latest/rhcos-4.3.8-x86_64-vmware.x86_64.ova]
	Command-line interface [wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz]

    Extract installer and oc commands, to a folder which is in your $PATH, in my case its ~/bin
    
	```	
    
	tar -xvf openshift-install-linux.tar.gz -c ~/bin
	tar -xvf openshift-client-linux.tar.gz -c ~/bin
    
	```
# Installation


* Step 1 - Generate and Upload Ignition

  ``` 
  cd ~
  git clone https://github.com/IBM-ICP4D/Cloud-Pak-for-Data-v3.0-Guide.git 
  cd Cloud-Pak-for-Data-v3.0-Guide/Installation/vsphere/install/
   
  ```
  
  Replace the values in the install-config.yaml file and make sure to take a backup of this file since this gets deleted during install.

  ``` 
  cp install-config.yaml install-config.yaml.bak
  openshift-install create ignition-configs
   
  ```
  
  The ignition and kubeconfig files are generated at this time. The kubeconfig file would be needed to authenticate with cluster later to communicate with API server.
  
  ```
  $ ls -al
  total 1580
	drwxr-xr-x. 3 root root     189 Apr 21 14:21 .
	drwxr-xr-x. 3 root root     257 Apr 20 04:28 ..
	drwxr-x---. 2 root root      80 Apr 20 17:07 auth
	-rw-r-----. 1 root root  292907 Apr 20 01:54 bootstrap.ign
	-rw-r--r--. 1 root root    3402 Apr 20 01:52 install-config.bak
	-rw-r-----. 1 root root    1825 Apr 20 01:54 master.ign
	-rw-r-----. 1 root root     112 Apr 20 01:54 metadata.json
	-rw-r-----. 1 root root    1825 Apr 20 01:54 worker.ign

   ```
   Copy the boostrap.ign to http server. In this case since HTTP server is on the same box 
   
   ``` 
   chmod 777 -R /var/www/html/
   
   cp bootstrap.ign /var/www/html/
   
   wget  http://lb.ocp4-cluster.cpd.xyz/bootstrap.ign should download the file. 
   
   ```
   
* Step 2 - Upload the Red Hat Enterprise Linux CoreOS (RHCOS) ova file on vsphere/install 
   
Follow the link to upload the rhcos ova file into the store as a default template. 
https://docs.vmware.com/en/VMware-vSphere/6.5/com.vmware.vsphere.vm_admin.doc/GUID-17BEDA21-43F6-41F4-8FB2-E01D275FE9B4.html
    
You could upload the file from the server or provide the URL of the location of ova file as provided earlier. The template must be named  as "rhcos-latest"
	
	Here is how it looks in vSphere after its uploaded
	
       

* Step 3 - Terraform configurations 

 Now that you have the image uploaded, ignition files generated, lets assemble all these into the terraform automation provided. 

```
 
$ cd ~
$ git clone https://github.com/openshift/installer.git
$ cd ~/installer/upi/vsphere
	
```
  
Copy the below content as terraform.tfvars and plug the values for vCenter details, ignition http details and static IPs. 
	
``` 
// ID identifying the cluster to create. Use your username so that resources created can be tracked back to you.
cluster_id = "ocp4-cluster"

// Domain of the cluster. This should be "${cluster_id}.${base_domain}".
cluster_domain = "ocp4-cluster.cpd.xyz"

// Base domain from which the cluster domain is a subdomain.
base_domain = "cpd.xyz"

// Name of the vSphere server. The dev cluster is on "vcsa.vmware.devcluster.openshift.com".
vsphere_server = "192.168.176.137"

// User on the vSphere server.
vsphere_user = "administrator@vsphere.local"

// Password of the user on the vSphere server.
vsphere_password = "*****"

// Name of the vSphere cluster. The dev cluster is "devel".
vsphere_cluster = "devel"

// Name of the vSphere data center. The dev cluster is "dc1".
vsphere_datacenter = "Datacenter"

// Name of the vSphere data store to use for the VMs. The dev cluster uses "nvme-ds1".
vsphere_datastore = "datastore3"

// Name of the VM template to clone to create VMs for the cluster. The dev cluster has a template named "rhcos-latest".
vm_template = "rhcos-latest"

// The machine_cidr where IP addresses will be assigned for cluster nodes.
// Additionally, IPAM will assign IPs based on the network ID.
machine_cidr = "192.168.176.0/24"

// The number of control plane VMs to create. Default is 3.
control_plane_count = 3

// The number of compute VMs to create. Default is 3.
compute_count = 2

// URL of the bootstrap ignition. This needs to be publicly accessible so that the bootstrap machine can pull the ignition.
bootstrap_ignition_url = "http://192.168.176.134/bootstrap.ign"

// Ignition config for the control plane machines. You should copy the contents of the master.ign generated by the installer.
control_plane_ignition = <<END_OF_MASTER_IGNITION
{"ignition":{"config":{"append":[{"source":"https://api-int.ocp4-cluster.cpd.xyz:22623/config/master....
END_OF_MASTER_IGNITION

// Ignition config for the compute machines. You should copy the contents of the worker.ign generated by the installer.
compute_ignition = <<END_OF_WORKER_IGNITION
{"ignition":{"config":{"append":[{"source":"https://api-int.ocp4-cluster.cpd.xyz:22623/config/worker",....
END_OF_WORKER_IGNITION


// Set ipam and ipam_token if you want to use the IPAM server to reserve IP
// addresses for the VMs.

// Address or hostname of the IPAM server from which to reserve IP addresses for the cluster machines.
ipam = "139.178.89.254"

// Token to use to authenticate with the IPAM server.
ipam_token = "TOKEN_FOR_THE_IPAM_SERVER"


// Set bootstrap_ip, control_plane_ip, and compute_ip if you want to use static
// IPs reserved someone else, rather than the IPAM server.

// The IP address to assign to the bootstrap VM.
bootstrap_ip = "192.168.176.100"

// The IP addresses to assign to the control plane VMs. The length of this list
// must match the value of control_plane_count.
control_plane_ips = ["192.168.176.101", "192.168.176.102", "192.168.176.103"]

// The IP addresses to assign to the compute VMs. The length of this list must
// match the value of compute_count.
compute_ips = ["192.168.176.104", "192.168.176.105" ]

```

 
Additional changes 

Modify the file main.tf and delete module "dns" 
Modify machine/ignition.tf if needed. In my case, I had to fix default gateway and DNS  to point it to the windows DNS server  

```

PREFIX=${local.mask}
#GATEWAY=${local.gw}
GATEWAY=192.168.176.2 (vmware specified gateway)
DNS1=10.148.168.144 
DNS2=8.8.8.8 (This was needed for installer to connect to internet for pulling images)

```

Modify variables.tf to adjust the number of nodes and possibly alter the size of each node.

	
* Step 4 - Provision machine and Install Openshift
 
```
$ cd ~/installer/upi/vsphere	
$ terraform init
$ terraform plan

```
If init and plan went well, the below command would start creating machines on the vsphere using the template. 
   
```
  $ terraform apply -auto-approve
```
Assuming that you have all ther permissions needed, 6 machines would created one by one and started. You should be able to see the events in the vpshere 

* Step 5 - Monitor the bootstrap process:

  ```
  $ cd Cloud-Pak-for-Data-v3.0-Guide/Installation/vsphere/install/
	$ openshift-install --dir=. wait-for bootstrap-complete --log-level=debug 

	INFO Waiting up to 30m0s for the Kubernetes API at https://api.test.example.com:6443...
	INFO API v1.14.6+c4799753c up
	INFO Waiting up to 30m0s for the bootstrap-complete event..  
        ........		
   
  ```
  
  
* Step 6 - After bootstrap process is complete, remove the bootstrap machine from the load balancer.

   Edit the below file and remove occurnces of backend bootstrap machine from machine-config-server and backend openshift-api-server sections
   restart haproxy as before and make sure its up and running 

* Step 7 - Wait for  Install to finish  

```
	
$ cd Cloud-Pak-for-Data-v3.0-Guide/Installation/vsphere/install/
$ openshift-install --dir=. wait-for bootstrap-complete --log-level=debug 

INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/Cloud-Pak-for-Data-v3.0-Guide/Installation/vsphere/install/auth/kubeconfig'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.ocp4-cluster.cpd.xyz/
INFO Login to the console with user: kubeadmin, password: *****	
	
```
  
* Step 8 - Login through oc command

```
	
$ cp ~/Cloud-Pak-for-Data-v3.0-Guide/Installation/vsphere/install/auth/kubeconfig ~/.kube/config
$ oc get nodes 
	
```
  ![alt text](https://github.com/IBM-ICP4D/Cloud-Pak-for-Data-v3.0-Guide/blob/master/Installation/vsphere/images/2020-04-21%2023_58_52.png "results")


  
