

![icp000](images/icp000.png)





![image-20190322173329630](images/image-20190322173329630-3272409.png)

# Multi-Node Cluster Installation Lab

The purpose of this lab is to install a complete IBM Cloud Private cluster running on multiple nodes.

> **Prerequisites** : you should have defined 4 (virtual) machines (RHEL version 7.6) and collected IP addresses and root passwords for these 4 machines.



---


# Task 1: Prerequisites

Before you start installing IBM Cloud Private, you must provision a set of four machines using your favorite vitualization tools or your prefered cloud infrastructure :
- One master node (including boot node, management node, va node, proxy node) : 8 vcores, 32 GB (RAM), 300 GB (Storage), one IP
- Three worker nodes:  8  vcores, 16 GB (RAM), 100 GB (Storage), one IP per worker
- One VM for NFS with NFS configured - or install the NFS server in the Master node
- All nodes are running RHEL version 7.6
- ICP Enterprise Edition - version 3.1.2

![image-20190514161953536](images/image-20190514161953536-7843593.png)



Let's say that we have a **prefix hostname** (**nicekkk** in my example). Then each node will get a complementary character or string at the end :
- m01 : example - nicekkkm01 for the master node (and all the other nodes included - see above)
- w01 or w02 or w030 : example - nicexkkkw01 and nicekkkw03 for the worker nodes
- The NFS node name is not involved at this point and it will be located on the master

**A list of Master VMs will be given to you during the workshop.** 

Please take one Master VM and put your name.

![image-20190514220017880](images/image-20190514220017880-7864017.png)

You only need the Master Node VM to start.

The name of each VM follows the following rules :

- all VMs start with **nice**
- 3 letters to make the cluster unique
- type of node : **m** for master and **w** for worker
- and finally a number on 2 digits like **01**, 02 ...

So, your list of VMs could be (for Paris) :

- niceparm01
- niceparw01
- niceparw02
- niceparw03

The prefix and the cluster name are the same value and this name is specified in the document for each participant.  The suffix is always '.ibm.ws'. And password is always admin1!

> **Please don't change the given prefix name, cluster name, cluster password and suffix name.**

All machines should be **up** and **running**. They also must be **accessible** to each other. 

Everything has been made with automation in mind. 



# Task 2: Step by step configuration

### Step 1: Login to the master machine

You must use <u>ssh</u> or <u>putty</u> to the **master node**.

**Login to your master**/boot node by using the following command (change with you master IP address) :

`ssh root@<masterip>`

Where the <masterip> is the ip address of your master (this has been given by the instructor)

You are going to stay on the Master node during all the exercises.

### Step 2: Setup the credentials for all machines

Prepare the following set of commands to set variables.

Normally you need to type all IPs and root passwords in a list like the one below (**please change the IPs and passwords, prefix and suffix accordingly to the ones you received**) :

```console
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
export CLUSTERPASS=admin1!
export SUFFIX=.ibm.ws
```

> Note that in this lab, all machines will be accessed using the super user **root**.
>
> CLUSTERNAME and PREFIX are identical.
>
> CLUSTERPASS=admin1!
>
> SUFFIX=.ibm.ws

Here is an example :

```console
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
export PREFIX=niceehnb
export CLUSTERNAME=niceehnb
export CLUSTERPASS=admin1!
export SUFFIX=.ibm.ws
```

**You don't need to type all these values** because we have prepared a file containing everything already set for you. 

Look at the the following file **icpinit**

`more icpinit`

This will normally produce the same list of variables. 

Check that we have set the variables by typing a command like: 

`echo $M1IP`

Results (as an example):

```
# echo $M1IP
158.176.128.214
```



###Step 3: Create a credential file for all your nodes

We need to create a credentials file for some next steps (to distribute automaticaly all keys and /etc/hosts to all nodes). 

Please **copy, paste and execute** the following commands in the master terminal:

```console
cd /root/.ssh
cat <<END > credentials
root:$W1PW@$W1IP
root:$W2PW@$W2IP
root:$W3PW@$W3IP
END
```

Check the output with the following command :

`more credentials`

Results (as an example):

```console
# more credentials 
root:DTs42t8a@158.176.122.11
root:D9xc5yvd@158.176.83.198
root:AK2xVbj8@158.176.122.38
```

###Step 4: Configure the /etc/hosts

We now need to create an **/etc/hosts** file for all nodes. **Execute** the following commands:

```console 
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

Check the results (as an example):

```console
# more /etc/hosts
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
158.176.122.50  niceehnbm01  niceehnbm01.ibm.ws
158.176.122.11  niceehnbw01  niceehnbw01.ibm.ws
158.176.83.198  niceehnbw02  niceehnbw02.ibm.ws
158.176.122.38  niceehnbw03  niceehnbw03.ibm.ws
```



#Task 3: Install Docker on the master node

First, you need to create a file that will be used to install Docker on the master and all the other  nodes. 

To do so execute the following command:
```console
cd /root

cat << 'END' > dockerinstall.sh
yum check-update -y
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce-18.03.1.ce-1.el7.centos
systemctl start docker
systemctl stop firewalld
systemctl disable firewalld
yum check-update -y
docker version
END
```

Then you can execute this script to install docker:
```console
cd /root
chmod +x dockerinstall.sh
./dockerinstall.sh
```

This command should end with the following results:
```console 
Client:
 Version:      18.03.1-ce
 API version:  1.37
 Go version:   go1.9.5
 Git commit:   9ee9f40
 Built:        Thu Apr 26 07:20:16 2018
 OS/Arch:      linux/amd64
 Experimental: false
 Orchestrator: swarm

Server:
 Engine:
  Version:      18.03.1-ce
  API version:  1.37 (minimum version 1.12)
  Go version:   go1.9.5
  Git commit:   9ee9f40
  Built:        Thu Apr 26 07:23:58 2018
  OS/Arch:      linux/amd64
  Experimental: false

```

#Task 4: Download ICP installation code

You can get the ICP zip file into the Passport Advantage IBM web site.

However, in your **/root** directory, we already have copied this zip file (**ibm-cloud-private-x86_64-3.1.2.tar.gz**):

```
# cd root
# ls -al
total 13046976
drwxr-xr-x. 2 root root        4096 Mar 22 04:33 .
drwxr-xr-x. 8 root root        4096 Mar 22 05:17 ..
-rw-r--r--. 1 root root 13347036806 Mar 21 18:41 ibm-cloud-private-x86_64-3.1.2.tar.gz
...
```

Execute the following commands:

```console
cd /root
tar xf ibm-cloud-private-x86_64-3.1.2.tar.gz -O | docker load
```

This unzips and loads docker images locally. It  can last **5 to 10 minutes**. Please wait and grab a coffee. 

#Task 5: Create ssh keys

On the master, we need to generate ssh keys that will copied across the cluster for **secure communications**. 

**Execute** the following commands:

```console 
mkdir /opt/icp
cd /opt/icp
docker run -v $(pwd):/data -e LICENSE=accept ibmcom/icp-inception-amd64:3.1.2-ee cp -r cluster /data

ssh-keygen -b 4096 -f ~/.ssh/id_rsa -N ""
cat ~/.ssh/id_rsa.pub | tee -a ~/.ssh/authorized_keys
systemctl restart sshd
cp -f ~/.ssh/id_rsa /opt/icp/cluster/ssh_key

```

> The result : id_rsa key is copied in the cluster directory.

Check the results:

```console
cd /opt/icp/cluster
[root@nicekkk01 cluster]# ls
cfc-certs  cfc-components  config.yaml  hosts logs  ssh_key

```

Move the ICP zip file into another location:

```console
cd /opt/icp
mkdir -p cluster/images
mv /root/ibm-cloud-private-x86_64-3.1.2.tar.gz  cluster/images/
```



### Distribute ssh keys on the other nodes

**Execute**  2 commands to **distribute** the keys and restart **sshd** on all nodes (these commands use sshpass that will transfer the pasword when requested):

```console
cd /root/.ssh

tr ':@' '\n' < credentials | xargs -L3 sh -c 'sshpass -p $1 ssh-copy-id -o StrictHostKeyChecking=no -f $0@$2'
tr ':@' '\n' < credentials | xargs -L3 sh -c 'ssh -o StrictHostKeyChecking=no $0@$2 systemctl restart sshd'
```

### Copy /etc/hosts on all nodes

**Execute** this command to copy remotely /etc/hosts on all worker nodes:

```console
tr ':@' '\n' < credentials | xargs -L3 sh -c 'scp -o StrictHostKeyChecking=no /etc/hosts $0@$2:/etc/hosts'
```

# Task 6: Configure ICP Installation

**Execute** this first command to define the ICP topology (one master and 2 workers). The management, the proxy and the VA (Vulnerability) are hosted in the master node:

```console 
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

Configure ICP file (config.yaml). To do so, **Execute** the following commands:

```console
cd /opt/icp/cluster
sed -i "s/# cluster_name: mycluster/cluster_name: $CLUSTERNAME/g" /opt/icp/cluster/config.yaml
sed -i 's/  vulnerability-advisor: disabled/  vulnerability-advisor: enabled/g' /opt/icp/cluster/config.yaml
sed -i 's/  istio: disabled/  istio: enabled/g' /opt/icp/cluster/config.yaml
sed -i "s/# default_admin_password:/default_admin_password: $CLUSTERPASS/g" /opt/icp/cluster/config.yaml
sed -i 's/# install_docker: true/install_docker: true/g' /opt/icp/cluster/config.yaml
echo "password_rules:" >> /opt/icp/cluster/config.yaml
echo "- '(.*)'" >> /opt/icp/cluster/config.yaml
```

> Note: These commands will configure config.yaml file to change the following:
>
> - cluster name is set
> - VA (Vulnerability Advisor) will be installed in the master node
> - Istio will be installed
> - Admin password is set (new !)
> - Docker will be installed automatically on the worker nodes
> - Password rule is set to anything (New and important !)



#Task 7: Install ICP Enterprise Edition

Type the following command to install ICP Cluster on the 3 nodes:

```console
cd /opt/icp/cluster
docker run --net=host -t -e LICENSE=accept -v "$(pwd)":/installer/cluster ibmcom/icp-inception-amd64:3.1.2-ee install
```
> Note : the installation should take **30 minutes**. So, if you don't see any error during the first 5 minutes, take another coffee or go to lunch. 

> Note : in case of error, you can retry the installation command **or** you can use the **uninstall** process :
```console 
cd /opt/icp/cluster
docker run --net=host -t -e LICENSE=accept -v "$(pwd)":/installer/cluster ibmcom/icp-inception-amd64:3.1.2-ee uninstall
```

At the end of the ICP installation, you will get the following results:

![image-20190322164525638](images/image-20190322164525638-3269525.png)

Take a note of the URL : `https://<masterip>:8443`

Check your connection to the master by using this URL (admin / admin1! ):

![image-20190322165536752](images/image-20190322165536752-3270136.png)

Got to the Dashboard: click on the hamburger **menu > Dashboard:**

![image-20190322165650422](images/image-20190322165650422-3270210.png)



# Task 8: CLI installations

**kubectl** has been already installed :

 ```console 
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

 The cloudctl CLI command should have been installed. **Execute** the following command **cloudctl**:

```console
# cloudctl
NAME:
   cloudctl - A command line tool to interact with IBM Cloud Private

USAGE:
[environment variables] cloudctl [global options] command [arguments...] [command options]

VERSION:
   3.1.1-973+c18caee2d82dc45146f843cb82ae7d5c28da7bc7

```

> cloudctl command is a usefull command to modify the infrastructure level of the cluster (like adding a new node or load helm repo). 



Modify the **connect2icp** command by **executing** the following commands:

```console
cd /root
sed -i "s/CLUSTERNAME=mycluster/CLUSTERNAME=$CLUSTERNAME/g" /root/connect2icp.sh
sed -i "s/PASSWD=admin/PASSWD=$CLUSTERPASS/g" /root/connect2icp.sh
chmod +x connect2icp.sh
./connect2icp.sh
```

> connect2icp.sh is a shell script to be executed after 12 hours (a token needs to renewed). This command tells kubectl where to connect.



Use the **cloudctl** command to **login** to the cluster (prerequisites before you install helm):

```console
cloudctl login -a https://$CLUSTERNAME.icp:8443 --skip-ssl-validation -u admin -p $CLUSTERPASS -n default
```

Results:

![image-20190322170842592](images/image-20190322170842592-3270922.png)



 Install helm with the following commands:

```console
cd
curl -O https://storage.googleapis.com/kubernetes-helm/helm-v2.9.1-linux-amd64.tar.gz
tar -vxhf helm-v2.9.1-linux-amd64.tar.gz
export HELM_HOME=/root/.helm
helm init --client-only
helm version --tls

docker login $CLUSTERNAME.icp:8500 -u admin -p $CLUSTERPASS
```

Results: 

```console
Configuring helm: /root/.helm
OK
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 8946k  100 8946k    0     0  24.4M      0 --:--:-- --:--:-- --:--:-- 24.4M
linux-amd64/
linux-amd64/README.md
linux-amd64/helm
linux-amd64/LICENSE
Creating /root/.helm/repository 
Creating /root/.helm/repository/cache 
Creating /root/.helm/repository/local 
Creating /root/.helm/plugins 
Creating /root/.helm/starters 
Creating /root/.helm/cache/archive 
Creating /root/.helm/repository/repositories.yaml 
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com 
Adding local repo with URL: http://127.0.0.1:8879/charts 
$HELM_HOME has been configured at /root/.helm.
Not installing Tiller due to 'client-only' flag having been set
Happy Helming!
Client: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.9.1+icp", GitCommit:"8ddf4db6a545dc609539ad8171400f6869c61d8d", GitTreeState:"clean"}
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
Login Succeeded

```

Now modify your .bashrc file so that when you login again, you will be connected to the cluster:

```console
cd
cat <<END >>.bashrc
./connect2icp.sh
export HELM_HOME=/root/.helm
cloudctl login -a https://$CLUSTERNAME.icp:8443 --skip-ssl-validation -u admin -p $CLUSTERPASS -n default
helm version --tls
END
source .bashrc
```



**Persistent volumes** : for our first installation, we need to have some basic hostpath volumes.

> Note that hostpath volumes will not be used in a HA production environment. 

 Type the following commands:

```console
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



# Task 9: Check your ICP console

Type the following URL in your browser:

`https://<masterip>:8443`

Go to the **menu** (top left), click on **Dashboard**

![image-20190322172231724](images/image-20190322172231724-3271751.png)

Check the number of nodes (3), the 48% of available storage and the 61 healthy pods.

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

- [x] You setup a VM using RHEL version 7.6

- [x] You checked all the prerequisites before the ICP installation

- [x] You installed Docker

- [x] You installed ICP Enterprise Edition (version 3.1.2) on a multi-node cluster

- [x] You connected to the ICP console

- [x] You setup some persistent storage

- [x] You installed a functional Kubernetes Cluster on a 3 nodes for testing and learning


# Appendix: "All in One" Script 

Find below one script file for the automated installation (All in one script - dont forget to change the IPs, passwords, clustername and prefix):

```console
#!/bin/bash
# ----------------------------------------- #
# PRODEDURE TO DEPLOY multiple Empty NODES  #
# ----------------------------------------- #
#
# Prerequisites
# -------------
#
# RHEL 7.6
# ICP 3.1.2 EE or CE
# IBM Cloud VSI
#  
# 
# Possible parameters:
#
# Global prefix :   VSI prefix name (SL only)
# DC name :         lon06 as default DataCenter name
# Edition :         EE or CE
#
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
export PREFIX=niceehnb
export CLUSTERNAME=niceehnb
export CLUSTERPASS=admin1!
export SUFFIX=.ibm.ws
#

##################################################################
# ICP 3.1.2 EE - Docker 18.03.1 - RHEL 7.6 - Automatic Docker
# One boot/Master/VA/Proxy, 3 Worksers
# Enterprise Edition
##################################################################

CYAN='\033[1;36m'
NC='\033[0m' # No Color

# Create Credentials (excluding Master) on the Master

echo -e "${CYAN}Create Credentials ${NC} "
cd /root/.ssh

cat <<END > credentials
root:$W1PW@$W1IP
root:$W2PW@$W2IP
root:$W3PW@$W3IP
END

# Create /etc/hosts on the Master

echo -e "${CYAN}Create hosts ${NC} "
cd /root

cat <<END > /etc/hosts
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4
$M1IP  ${PREFIX}m01  ${PREFIX}m01${SUFFIX}
$W1IP  ${PREFIX}w01  ${PREFIX}w01${SUFFIX}
$W2IP  ${PREFIX}w02  ${PREFIX}w02${SUFFIX}
$W3IP  ${PREFIX}w03  ${PREFIX}w03${SUFFIX}
END

# Create Docker installation script on the Master

echo -e "${CYAN}Create Docker installation script ${NC} "
cd /root

cat << 'END' > dockerinstall.sh
yum check-update -y
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce-18.03.1.ce-1.el7.centos
systemctl start docker
systemctl stop firewalld
systemctl disable firewalld
yum check-update -y
docker version
END


# Install Docker on the Master

echo -e "${CYAN}Install Docker on Master ${NC}"
cd /root
chmod +x dockerinstall.sh
./dockerinstall.sh


# Upload & unzip ICP installation file to the Master (takes 10mn)

echo -e "${CYAN}Unzip ICP installation file on Master ${NC}"
cd /root
tar xf ibm-cloud-private-x86_64-3.1.2.tar.gz -O | docker load


# Copy Config Files for icp-ee in /opt directory on the Master

echo -e "${CYAN}Copy Config Files on Master ${NC}"
cd /root
mkdir /opt/icp
cd /opt/icp
docker run -v $(pwd):/data -e LICENSE=accept ibmcom/icp-inception-amd64:3.1.2-ee cp -r cluster /data


# Create Keys in /opt/icp on the Master

echo -e "${CYAN}Create Keys on Master ${NC}"
ssh-keygen -b 4096 -f ~/.ssh/id_rsa -N ""
cat ~/.ssh/id_rsa.pub | tee -a ~/.ssh/authorized_keys
systemctl restart sshd
cp -f ~/.ssh/id_rsa /opt/icp/cluster/ssh_key


# Save the installation zip file to images

echo -e "${CYAN}Save the installation zip file to images${NC}"
cd /opt/icp
mkdir -p cluster/images
mv /root/ibm-cloud-private-x86_64-3.1.2.tar.gz  cluster/images/


# From the master - Install Hosts, Docker, restart SSH and copy keys on each NODE

echo -e "${CYAN}Install Hosts, Docker, restart SSH and copy keys on each NODE${NC}"
cd /root/.ssh
tr ':@' '\n' < credentials | xargs -L3 sh -c 'sshpass -p $1 ssh-copy-id -o StrictHostKeyChecking=no -f $0@$2'
tr ':@' '\n' < credentials | xargs -L3 sh -c 'ssh -o StrictHostKeyChecking=no $0@$2 systemctl restart sshd'
tr ':@' '\n' < credentials | xargs -L3 sh -c 'scp -o StrictHostKeyChecking=no /etc/hosts $0@$2:/etc/hosts'


# Customize hosts before installing ICP

echo -e "${CYAN}Customize hosts before installing ICP ${NC}"
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

echo -e "${CYAN}Configure ICP config.yaml file ${NC}"
cd /opt/icp/cluster
sed -i "s/# cluster_name: mycluster/cluster_name: $CLUSTERNAME/g" /opt/icp/cluster/config.yaml
sed -i 's/  vulnerability-advisor: disabled/  vulnerability-advisor: enabled/g' /opt/icp/cluster/config.yaml
sed -i 's/  istio: disabled/  istio: enabled/g' /opt/icp/cluster/config.yaml
sed -i "s/# default_admin_password:/default_admin_password: $CLUSTERPASS/g" /opt/icp/cluster/config.yaml
sed -i 's/# install_docker: true/install_docker: true/g' /opt/icp/cluster/config.yaml
echo "password_rules:" >> /opt/icp/cluster/config.yaml
echo "- '(.*)'" >> /opt/icp/cluster/config.yaml


# Installation ICP-EE

echo -e "${CYAN}Installation ICP-EE ${NC}"
cd /opt/icp/cluster
docker run --net=host -t -e LICENSE=accept -v "$(pwd)":/installer/cluster ibmcom/icp-inception-amd64:3.1.2-ee install


# Modify connect2icp to support clustername and password

echo -e "${CYAN}Modify connect2icp ${NC}"
cd /root
sed -i "s/CLUSTERNAME=mycluster/CLUSTERNAME=$CLUSTERNAME/g" /root/connect2icp.sh
sed -i "s/PASSWD=admin/PASSWD=$CLUSTERPASS/g" /root/connect2icp.sh
chmod +x connect2icp.sh
./connect2icp.sh
cloudctl login -a https://$CLUSTERNAME.icp:8443 --skip-ssl-validation -u admin -p $CLUSTERPASS -n default


# Installing HELM

echo -e "${CYAN}Installing HELM ${NC}"
cd /root
curl -O https://storage.googleapis.com/kubernetes-helm/helm-v2.9.1-linux-amd64.tar.gz
tar -vxhf helm-v2.9.1-linux-amd64.tar.gz
export PATH=/root/linux-amd64:$PATH
export HELM_HOME=/root/.helm
helm init --client-only
helm version --tls
docker login $CLUSTERNAME.icp:8500 -u admin -p $CLUSTERPASS

# Source .bashrc

cat <<END >>.bashrc
./connect2icp.sh
export HELM_HOME=/root/.helm
cloudctl login -a https://$CLUSTERNAME.icp:8443 --skip-ssl-validation -u admin -p $CLUSTERPASS -n default
helm version --tls
END
source .bashrc

# Installing Persistent Storage in the Master

echo -e "${CYAN}Installing Persistent Storage in the Master ${NC}"
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

echo -e "${CYAN}End of Installation ${NC}"
echo -e "${CYAN}MASTERIP is : $MASTERIP ${NC}"


# End
date



```

![icp000](images/icp000.png)

