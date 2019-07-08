

![icp000](images/icp000.png)





![image-20190322173329630](images/image-20190322173329630-3272409.png)

# Multi-Node Cluster Installation Lab

The purpose of this lab is to install a complete IBM Cloud Private cluster running on multiple nodes.

> **Note** : you should have defined 4 (virtual) machines (RHEL version 7.6) and collected IP addresses and root passwords for these machines.



---


# Task 1: Prerequisites

Before you start installing IBM Cloud Private, you must provision a set of four machines using your favorite vitualization tools or on your prefered cloud infrastructure :
- One master node (including boot node, management node, va node, proxy node) : 8 vcores, 32 GB (RAM), 300 GB (Storage), one IP
- Three worker nodes:  8  vcores, 16 GB (RAM), 100 GB (Storage), one IP per worker
- All nodes are running RHEL version 7.6
- ICP Enterprise Edition - version 3.2

Concerning NFS, in that case,  we will install the NFS server later in the Master node (see a specific lab for this).

![image-20190514161953536](images/image-20190514161953536-7843593.png)



Let's say that we have a **prefix hostname** (**nicekkk** in my example). Then each node will get a complementary character or string at the end :
- m01 : example - nicekkkm01 for the master node (and all the other nodes included - see above)
- w01 or w02 or w03 : for example - nicexkkkw01,  nicexkkkw02 and nicekkkw03 for the worker nodes
- The NFS node name is not involved at this point and it will be located on the master



A list of Master VMs will be given to you during the workshop.

Please pick one Master VM and write your name on the right part of the document.

![image-20190514220017880](images/image-20190514220017880-7864017.png)

You only need to know the Master Node VM to work on the labs.

The name of each VM follows the following conventions:

- all VMs start with **nice**
- **3 letters** to make the cluster unique (kkk in my example)
- the type of node : **m** for master and **w** for worker
- and finally, 2 digits like **01**, 02 ...

So, your list of VMs could be for example:

- niceparm01
- niceparw01
- niceparw02
- niceparw03

The prefix and the cluster name are the same value and this name is specified in the document for each participant.  The suffix is always '.ibm.ws'. 

There are 2 passwords : one to ssh to the VM and the other to access the cluster. This second value is accessible thru `echo $CLUSTERPASS`

> **Please don't change the given prefix name, cluster name, cluster password and suffix name.**

All machines should be **up** and **running**. 

They also must be **accessible** to each other thru their IPs. 

Everything has been made with automation in mind. 



# Task 2: Step by step configuration

### Step 1: Login to the master machine

You must use <u>ssh</u> or <u>putty</u> to the **master node**.

**Login to your master** node by using the following command (change with you master IP address) :

```bash
ssh root@<masterip>
```

Where the <masterip> is the ip address of your master (this has been given by the instructor in the document)

You will stay on the Master node during all the labs.

### Step 2: Setup the credentials for all machines

We have decided to automate most of the installation by setting **variables**.

Normally you need to type all IPs and root passwords in a list of variables like the one below:

```bash
export M1N=
export M1IP=
export M1PW=
export W1N=
export W1IP=
export W1PW=
export W2N=
export W2IP=
export W2PW=
export W3N=
export W3IP=
export W3PW=
export PREFIX=
export CLUSTERNAME=
export CLUSTERPASS=
export SUFFIX=.ibm.ws
```

**We have already prepared files containing these variables for you.** All variables are set thru .bashrc or in the icpinit files. 

Look at the the following file **icpinit**:

```bash
more icpinit
```

Results:

```bash
export M1N=niceehnbm01
export M1IP=158.176.122.50
export M1PW=Hy28n7aC
export W1N=niceehnbw01
export W1IP=158.176.122.11
export W1PW=DTs42t8a
export W2N=niceehnbw02
export W2IP=158.176.83.198
export W2PW=D9xc5yvd
export W3N=niceehnbw03
export W3IP=158.176.122.38
export W3PW=AK2xVbj8
export PREFIX=nicekkk
export CLUSTERNAME=nicekkk
export CLUSTERPASS=admin1!
export SUFFIX=.ibm.ws
```

Check that all the variables have a value.

Alternatively, type a command like: 

`echo $M1IP`

Results (as an example):

```bash
# echo $M1IP
158.176.128.214
```

> **Don't change the values or the variable names**.
>
> All the steps and most labs are using these variables.



###Step 3: Create a credential file for all your nodes

We need to create a credential file for some next steps (to distribute automaticaly all keys and /etc/hosts to all nodes). 

Please **copy, paste and execute** the following commands in the master terminal:

```bash
cd /root/.ssh
cat <<END > credentials
root:$W1PW@$W1IP
root:$W2PW@$W2IP
root:$W3PW@$W3IP
END
```

Check the output with the following command :

```bash
more credentials
```

Results (as an example):

```bash
# more credentials 
root:DTs42t8a@158.176.122.11
root:D9xc5yvd@158.176.83.198
root:AK2xVbj8@158.176.122.38
```

###Step 4: Configure the /etc/hosts

We now need to create an **/etc/hosts** file for all nodes. **Execute** the following commands:

```bash
cd /root
cat <<END > /etc/hosts
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
$M1IP  ${PREFIX}m01  ${PREFIX}m01${SUFFIX}
$W1IP  ${PREFIX}w01  ${PREFIX}w01${SUFFIX}
$W2IP  ${PREFIX}w02  ${PREFIX}w02${SUFFIX}
$W3IP  ${PREFIX}w03  ${PREFIX}w03${SUFFIX}
END
```
> Note : the dns suffix name used here is .ibm.ws could not be change for this lab.

```bash
more /etc/hosts
```

Check the results (as an example):

```bash
# more /etc/hosts
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
158.176.122.50  niceehnbm01  niceehnbm01.ibm.ws
158.176.122.11  niceehnbw01  niceehnbw01.ibm.ws
158.176.83.198  niceehnbw02  niceehnbw02.ibm.ws
158.176.122.38  niceehnbw03  niceehnbw03.ibm.ws
```



#Task 3: Install Docker on the master node

First, you need to create a file that will be used to install Docker on the master (a file containing docker installation **icp-docker-18.06.2_x86_64.bin** has been already uploaded on your master). You can also install Docker from Docker Hub (version 18.06.2 is required).

To do so,  **execute** the following commands:
```bash
cd /root
cat << 'END' > dockerinstall.sh
chmod +x icp-docker-18.06.2_x86_64.bin
./icp-docker-18.06.2_x86_64.bin --install
systemctl start docker
systemctl stop firewalld
systemctl disable firewalld
yum check-update -y
docker version
END
```

Then you can **execute** this script to install docker:
```bash
cd /root
chmod +x dockerinstall.sh
./dockerinstall.sh
```

This command should end with the following results:
```bash
...
Client:
 Version:           18.06.2-ce
 API version:       1.38
 Go version:        go1.10.3
 Git commit:        %{_gitcommit}
 Built:             Mon Mar 25 06:03:28 2019
 OS/Arch:           linux/amd64
 Experimental:      false

Server:
 Engine:
  Version:          18.06.2-ce
  API version:      1.38 (minimum version 1.12)
  Go version:       go1.10.3
  Git commit:       %{_gitcommit}
  Built:            Mon Mar 25 06:04:45 2019
  OS/Arch:          linux/amd64
  Experimental:     false


```



#Task 4: Download ICP installation code

You can get the ICP zip file into the Passport Advantage IBM web site.

However, in your **/root** directory, we already have copied this zip file (**ibm-cloud-private-x86_64-3.2.0.tar.gz**) for you.

```
cd /root
ls -al
```

Results:

```bash
# cd root
# ls -al
total 13046976
drwxr-xr-x. 2 root root        4096 Mar 22 04:33 .
drwxr-xr-x. 8 root root        4096 Mar 22 05:17 ..
-rw-r--r--. 1 root root 10697215684 Jul 7 18:41 ibm-cloud-private-x86_64-3.2.0.tar.gz
...
```

**Execute** the following commands:

```bash
cd /root
tar xf ibm-cloud-private-x86_64-3.2.0.tar.gz -O | docker load
```

This unzips and loads docker images locally in the master. 

It  can last **10 minutes**. Please wait and grab a coffee. 



#Task 5: Create ssh keys

On the master, we need to generate ssh keys that we will copy across the cluster for **secure communications**. 

**Execute** the following commands:

```bash
cd /root
mkdir /opt/icp
cd /opt/icp
docker run -v $(pwd):/data -e LICENSE=accept ibmcom/icp-inception-amd64:3.2.0-ee cp -r cluster /data

ssh-keygen -b 4096 -f ~/.ssh/id_rsa -N ""
cat ~/.ssh/id_rsa.pub | tee -a ~/.ssh/authorized_keys
systemctl restart sshd
cp -f ~/.ssh/id_rsa /opt/icp/cluster/ssh_key

```

> The result : id_rsa key is copied in the cluster directory.

Check the results:

```bash
cd /opt/icp/cluster
# ls
config.yaml  hosts logs  ssh_key
```

**Move** the ICP zip file into another location:

```bash
cd /opt/icp
mkdir -p cluster/images
mv /root/ibm-cloud-private-x86_64-3.2.0.tar.gz  cluster/images/
```



### Distribute ssh keys on the other nodes

**Execute**  those commands to **distribute** the keys to all nodes and restart **sshd** on all nodes (these commands use sshpass that will type the pasword when requested):

```bash
cd /root/.ssh

tr ':@' '\n' < credentials | xargs -L3 sh -c 'sshpass -p $1 ssh-copy-id -o StrictHostKeyChecking=no -f $0@$2'
tr ':@' '\n' < credentials | xargs -L3 sh -c 'ssh -o StrictHostKeyChecking=no $0@$2 systemctl restart sshd'
```

After this step, we will no longer need passwords to access worker nodes.



### Copy /etc/hosts on all nodes

**Execute** this command to copy remotely /etc/hosts on all worker nodes:

```bash
tr ':@' '\n' < credentials | xargs -L3 sh -c 'scp -o StrictHostKeyChecking=no /etc/hosts $0@$2:/etc/hosts'
```

Then we will use hostnames to get access to worker nodes instead of IP addresses.



# Task 6: Configure ICP Installation

**Execute** this code to define the ICP topology (one master and 3 workers). The management, the proxy and the VA (Vulnerability) are hosted in the master node:

```bash
cd /root
cat <<END > /opt/icp/cluster/hosts
[master]
$M1IP
[worker]
$W1IP
$W2IP
$W3IP
[proxy]
$M1IP
[management]
$M1IP
[va]
$M1IP
END
```

Configure the **config.yaml** file To do so, **Execute** the following commands:

```bash
cd /opt/icp/cluster
sed -i "s/# cluster_name: mycluster/cluster_name: $CLUSTERNAME/g" /opt/icp/cluster/config.yaml
sed -i 's/  vulnerability-advisor: disabled/  vulnerability-advisor: enabled/g' /opt/icp/cluster/config.yaml
sed -i "s/# default_admin_password:/default_admin_password: $CLUSTERPASS/g" /opt/icp/cluster/config.yaml
sed -i 's/# install_docker: true/install_docker: true/g' /opt/icp/cluster/config.yaml
echo "password_rules:" >> /opt/icp/cluster/config.yaml
echo "- '(.*)'" >> /opt/icp/cluster/config.yaml
```

> Note: These commands will configure config.yaml file to change the following:
>
> - cluster name is set
> - VA (Vulnerability Advisor) will be installed in the master node
> - Admin password is set 
> - Docker will be installed automatically on the worker nodes
> - Password rule is set to anything (New and important !)



To understand the installation setup, you can look at the **config.yaml** file where you have all the parameters:

```bash
nano /opt/icp/cluster/config.yaml
```

Review some options (**but don't change any parameters**) :

```
# Licensed Materials - Property of IBM
# IBM Cloud private
# @ Copyright IBM Corp. 2017 All Rights Reserved
# US Government Users Restricted Rights - Use, duplication or disclosure restricted by GSA ADP Schedule Contract with IBM Corp.

---

## Network Settings
network_type: calico
# network_helm_chart_path: < helm chart path >

## Network in IPv4 CIDR format
network_cidr: 10.1.0.0/16

## Kubernetes Settings
service_cluster_ip_range: 10.0.0.0/16

# cluster_domain: cluster.local
# cluster_name: mycluster
# cluster_CA_domain: "{{ cluster_name }}.icp"

...
```



#Task 7: Install ICP Enterprise Edition

Type the following command to install ICP Cluster on the 4 nodes:

```console
cd /opt/icp/cluster
docker run --net=host -t -e LICENSE=accept -v "$(pwd)":/installer/cluster ibmcom/icp-inception-amd64:3.2.0-ee install
```
> Note : the installation should take **35 minutes**. So, if you don't see any error during the first 5 minutes, take another coffee or go to lunch. 



At the end of the ICP installation, you will get the following results:

![image-20190322164525638](images/image-20190322164525638-3269525.png)

Take a note of the URL : `https://<masterip>:8443`

> **Important Note** : in case of error, you can either **retry** the installation command **or** you can use the **uninstall** process (it takes generally 2 minutes) :

```console 
cd /opt/icp/cluster
docker run --net=host -t -e LICENSE=accept -v "$(pwd)":/installer/cluster ibmcom/icp-inception-amd64:3.2.0-ee uninstall
```

Check your connection to the master by using the URL

![image-20190322165536752](images/image-20190322165536752-3270136.png)

Got to the Dashboard: click on the hamburger **menu > Dashboard:**

![image-20190322165650422](images/image-20190322165650422-3270210.png)



# Task 8: CLI installations

**kubectl** has been already installed on the master node :

```bash
kubectl
```

Results:

 ```bash
# kubectl
kubectl controls the Kubernetes cluster manager. 

Find more information at: https://kubernetes.io/docs/reference/kubectl/overview/

Basic Commands (Beginner):
  create         Create a resource from a file or from stdin.
  expose         Take a replication controller, service, deployment or pod and expose it as a new Kubernetes Service
  run            Run a particular image on the cluster
  set            Set specific features on objects

Basic Commands (Intermediate) ...
 ```

 

**cloudctl** CLI command should have been also installed. 

```bash
cloudctl
```

Results:

```bash
# cloudctl
NAME:
   cloudctl - A command line tool to interact with IBM Cloud Private

USAGE:
[environment variables] cloudctl [global options] command [arguments...] [command options]

VERSION:
   3.1.1-973+c18caee2d82dc45146f843cb82ae7d5c28da7bc7
...
```

> cloudctl command is a usefull command to modify the infrastructure level of the cluster (like adding a new node or load helm repo). 



Use the **cloudctl** command to login to the cluster (prerequisites before you install helm):

```bash
cloudctl login -a https://$CLUSTERNAME.icp:8443 --skip-ssl-validation -u admin -p $CLUSTERPASS -n default
```

Results (as an example):

```bash
# cloudctl login -a https://mycluster.icp:8443 --skip-ssl-validation -u admin -p admin1! -n default
Authenticating...
OK

Targeted account mycluster Account (id-mycluster-account)

Targeted namespace default

Configuring kubectl ...
Property "clusters.mycluster" unset.
Property "users.mycluster-user" unset.
Property "contexts.mycluster-context" unset.
Cluster "mycluster" set.
User "mycluster-user" set.
Context "mycluster-context" created.
Switched to context "mycluster-context".
OK

Configuring helm: /root/.helm
OK
```



Check **helm** with the following commands:

```bash
cd
helm version --tls --short
```

Results: 

```bash
# helm version --tls --short
Client: v2.12.3+geecf22f
Server: v2.12.3+icp+g34e12ad
```



Get access to the **registry**:

```
docker login $CLUSTERNAME.icp:8500 -u admin -p $CLUSTERPASS
```

Results:

```bash
# docker login $CLUSTERNAME.icp:8500 -u admin -p $CLUSTERPASS
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```



Now modify your **.bashrc** file so that when you login again, you will be connected to the cluster:

```bash
cd
cat <<END >>.bashrc
export HELM_HOME=/root/.helm
# cloudctl login -a https://$CLUSTERNAME.icp:8443 --skip-ssl-validation -u admin -p $CLUSTERPASS -n default
helm version --tls
docker login $CLUSTERNAME.icp:8500 -u admin -p $CLUSTERPASS
END
source .bashrc
```



**Persistent volumes** : for our first installation, we need to have some basic hostpath volumes.

> Note that hostpath volumes will not be used in a HA production environment. 

 Type the following commands:

```bash
cd /var
mkdir data01

cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv-once-test1
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 50Gi
  hostPath:
    path: /var/data01
  persistentVolumeReclaimPolicy: Recycle
EOF

cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv-many-test1
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 50Gi
  hostPath:
    path: /var/data01
  persistentVolumeReclaimPolicy: Recycle
EOF

cd /root
```



# Task 9: Using the IBM Cloud Private UI

Type the following URL in your browser:

```http
https://<masterip>:8443
```



Go to the **menu** (top left), click on **Dashboard**

![image-20190322172231724](images/image-20190322172231724-3271751.png)

Check the number of nodes (4), around 48% of available storage and the 0 unhealthy pods.

Go to the **menu** (top left), click on **Platform** then on **Storage**:

![image-20190322172456003](images/image-20190322172456003-3271896.png)

At this point you should see your 2 new Hostpath Persistent storage.

Click on the **Catalog** menu (top right) to look at the list of applications already installed.

The Catalog shows Charts that you can visit (it could take au few seconds to refresh the first time)

You can look at the (helm) catalog and visit some entries (but don't create any application at the moment).

![image-20190322182137396](images/image-20190322182137396-3275297.png)





# Congratulations 

You have successfully installed, deployed and customized the Kubernetes Cluster for an **IBM Cloud Private Enterprise Edition** !

You finally went thru the following features :

- [x] You setup a cluster of VMs using RHEL version 7.6

- [x] You checked all the prerequisites before the ICP installation

- [x] You installed Docker

- [x] You installed ICP Enterprise Edition (version 3.2) on a multi-node cluster

- [x] You connected to the ICP console

- [x] You setup some persistent storage

- [x] You installed a functional Kubernetes Cluster on a 4 nodes for testing and learning



# Appendix: "All in One" Script 

Find below one script file for the automated installation (All in one script - dont forget to change the IPs, passwords, clustername and prefix).

Follow the steps :

- copy the lines into a file like **icpinstall.sh** in you /root directory
- edit the file and change the **IPs, passwords, clustername and prefix** from the icpinit file or your own requirements
- change the permission: `chmod +x icpinstall.sh`
- Execute the script: `./icpinstall.sh`
- Don't forget to continue on task 8



**Example of automated installation script**

```bash
#!/bin/bash
# 
export M1N=nicebotm01
export M1IP=158.176.128.155
export M1PW=BWQP7n3U
export W1N=nicebotw01
export W1IP=158.176.128.167
export W1PW=CNczLh7X
export W2N=nicebotw02
export W2IP=158.176.128.139
export W2PW=Rear7BPr
export W3N=nicebotw03
export W3IP=158.176.128.219
export W3PW=EJ3xaxQx
export PREFIX=nicebot
export CLUSTERNAME=nicebot
export CLUSTERPASS=nicebot-
export SUFFIX=.ibm.ws
#

##################################################################
# ICP 3.2.0 EE - Docker 18.06.2 - RHEL 7.6 - Automatic Docker
# One boot/Master/VA/Proxy, 3 Worksers
# Enterprise Edition
##################################################################

CC='\033[1;36m'
NC='\033[0m' # No Color

# Create Credentials (excluding Master) on the Master

echo -e "$CC Create Credentials $NC"
cd /root/.ssh

cat <<END > credentials
root:$W1PW@$W1IP
root:$W2PW@$W2IP
root:$W3PW@$W3IP
END

# Create /etc/hosts on the Master

echo -e "$CC Create hosts $NC"
cd /root

cat <<END > /etc/hosts
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
$M1IP  ${PREFIX}m01  ${PREFIX}m01${SUFFIX}
$W1IP  ${PREFIX}w01  ${PREFIX}w01${SUFFIX}
$W2IP  ${PREFIX}w02  ${PREFIX}w02${SUFFIX}
$W3IP  ${PREFIX}w03  ${PREFIX}w03${SUFFIX}
END

# Create Docker installation script on the Master

echo -e "$CC Create Docker installation script $NC"
cd /root

cat << 'END' > dockerinstall.sh
yum check-update -y
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
chmod +x icp-docker-18.06.2_x86_64.bin
./icp-docker-18.06.2_x86_64.bin --install
systemctl start docker
systemctl stop firewalld
systemctl disable firewalld
yum check-update -y
docker version
END


# Install Docker on the Master

echo -e "$CC Install Docker on Master $NC"
cd /root
chmod +x dockerinstall.sh
./dockerinstall.sh


# Upload & unzip ICP installation file to the Master (takes 10mn)

echo -e "$CC Unzip ICP installation file on Master $NC"
cd /root
tar xf ibm-cloud-private-x86_64-3.2.0.tar.gz -O | docker load


# Copy Config Files for icp-ee in /opt directory on the Master

echo -e "$CC Copy Config Files on Master $NC"
cd /root
mkdir /opt/icp
cd /opt/icp
docker run -v $(pwd):/data -e LICENSE=accept ibmcom/icp-inception-amd64:3.2.0-ee cp -r cluster /data


# Create Keys in /opt/icp on the Master

echo -e "$CC Create Keys on Master $NC"
ssh-keygen -b 4096 -f ~/.ssh/id_rsa -N ""
cat ~/.ssh/id_rsa.pub | tee -a ~/.ssh/authorized_keys
systemctl restart sshd
\cp -f ~/.ssh/id_rsa /opt/icp/cluster/ssh_key


# Save the installation zip file to images

echo -e "$CC Save the installation zip file to images$NC"
cd /opt/icp
mkdir -p cluster/images
mv /root/ibm-cloud-private-x86_64-3.2.0.tar.gz  cluster/images/


# From the master - Install Hosts, Docker, restart SSH and copy keys on each NODE

echo -e "$CC Install Hosts, Docker, restart SSH and copy keys on each NODE$NC"
cd /root/.ssh
tr ':@' '\n' < credentials | xargs -L3 sh -c 'sshpass -p $1 ssh-copy-id -o StrictHostKeyChecking=no -f $0@$2'
tr ':@' '\n' < credentials | xargs -L3 sh -c 'ssh -o StrictHostKeyChecking=no $0@$2 systemctl restart sshd'
tr ':@' '\n' < credentials | xargs -L3 sh -c 'scp -o StrictHostKeyChecking=no /etc/hosts $0@$2:/etc/hosts'


# Customize hosts before installing ICP

echo -e "$CC Customize hosts before installing ICP $NC"
cd /root
cat <<END > /opt/icp/cluster/hosts
[master]
$M1IP
[worker]
$W1IP
$W2IP
$W3IP
[proxy]
$M1IP
[management]
$M1IP
[va]
$M1IP
END

# Configure ICP config.yaml file

echo -e "$CC Configure ICP config.yaml file $NC"
cd /opt/icp/cluster
sed -i "s/# cluster_name: mycluster/cluster_name: $CLUSTERNAME/g" /opt/icp/cluster/config.yaml
sed -i 's/  vulnerability-advisor: disabled/  vulnerability-advisor: enabled/g' /opt/icp/cluster/config.yaml
sed -i "s/# default_admin_password:/default_admin_password: $CLUSTERPASS/g" /opt/icp/cluster/config.yaml
sed -i 's/# install_docker: true/install_docker: true/g' /opt/icp/cluster/config.yaml
echo "password_rules:" >> /opt/icp/cluster/config.yaml
echo "- '(.*)'" >> /opt/icp/cluster/config.yaml


# Installation ICP-EE

echo -e "$CC Installation ICP-EE $NC"
cd /opt/icp/cluster
docker run --net=host -t -e LICENSE=accept -v "$(pwd)":/installer/cluster ibmcom/icp-inception-amd64:3.2.0-ee install
cloudctl login -a https://$CLUSTERNAME.icp:8443 --skip-ssl-validation -u admin -p $CLUSTERPASS -n default

# check HELM

echo -e "$CC Installing HELM $NC"
cd /root
export PATH=/root/linux-amd64:$PATH
export HELM_HOME=/root/.helm
helm init --client-only
helm version --tls --short
docker login $CLUSTERNAME.icp:8500 -u admin -p $CLUSTERPASS

# Installing Persistent Storage in the Master

echo -e "$CC Installing Persistent Storage in the Master $NC"
cd /tmp
mkdir data01

cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv-once-test1
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 50Gi
  hostPath:
    path: /tmp/data01
  persistentVolumeReclaimPolicy: Recycle
EOF

cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv-many-test1
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 50Gi
  hostPath:
    path: /tmp/data01
  persistentVolumeReclaimPolicy: Recycle
EOF

cd /root

echo -e "$CC End of Installation $NC"
echo -e "$CC MASTERIP is : $MASTERIP $NC"


# End


```

![icp000](images/icp000.png)

