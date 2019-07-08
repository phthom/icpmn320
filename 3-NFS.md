![icp000](images/icp000.png)



![image-20190329160530064](images/image-20190329160530064-3871930.png)



# NFS persistent Storage with IBM Cloud Private



Network File System (NFS), due to its simplicity and the well-established techniques, are commonly available in the Enterprise IT infrastructure.

Kubernetes support the NFS based Persistent Volume (PV). 

This document explains the steps to enable the NFS dynamic volume provisioning in IBM Cloud Private (ICP) 3.2, and some simple validation test.

### 1. Configuring a NFS server

Find below some basic instructions to setup and configure a NFS server. 

We are going to configure a NFS server in the **Master node** (normally we use specific servers for NFS but in that case this is for testing only).

- Install a specific server with RHEL version 7 and some storage (in this case, we choose the **Master node** for simplicity)

- Install **nfs-utils** (below an example on RHEL) or the **nfs-common** (Ubuntu)

`yum -y install nfs-utils`

- Create a data directory

`mkdir /data`

- Change permissions (important)

`chmod 777 /data`

- Configure the following NFS server file :

`nano /etc/exports`

Content of the exports file

```console
# After modif
#systemctl restart nfs-config
#systemctl restart nfs-server
#repetoire  host(option)
/data remoteip(rw,async,no_root_squash) 
```

Where /data is your shared directory and remoteip is the ip address of your kubernetes nodes. You need to add all the ip addresses (and options) for **all workers** in the cluster in a **row**. You can get all your worker ip in the icpinit or at the end of the .bashrc file.

For example :

```console
# Dont forget to do this 2 commands after modif
#systemctl restart nfs-config
#systemctl restart nfs-server
#directory  host(option)
/data 158.176.122.25(rw,async,no_root_squash) 158.176.122.27(rw,async,no_root_squash) 158.176.122.24(rw,async,no_root_squash)
```

Save your change (Ctrl O + enter + Ctrl X)

Then don't forget also to **restart** both the nfs-config and nfs-server:

```
systemctl restart nfs-config
systemctl restart nfs-server
```

Results:

```console
# systemctl restart nfs-config
# systemctl restart nfs-server
```



## 2. Enable Image Security in ICP

Be default ICP 3.2 enforce the image security allowing only those images that are defined in a white list to run in the cluster.

Execute the following object,

```
kubectl create -f - <<ZZZ
apiVersion: securityenforcement.admission.cloud.ibm.com/v1beta1
kind: ClusterImagePolicy
metadata:
  name: my-cluster-images-nfs-client
spec:
  repositories:
    - name: quay.io/external_storage/*
ZZZ
```



## 3. Deploy the NFS Client Provisioner Helm Chart

Run the helm command to install the helm chart with the following command

```
helm repo update

helm install --name nfs-provisioner --set podSecurityPolicy.enabled=true --set nfs.server=<ipaddressNFSserver> --set nfs.path=/data stable/nfs-client-provisioner --tls
```

Supply the NFS server’s IP address/hostname, and the exported NFS path (/data in that case). We also enable the flag *podSecurityPolicy.enabled* which is required for ICP 3.1.2. *(For production usage, you may want to define a name instead of a randomly generated release name)*. Storage class is : nfs-client.

Watch the pod is running, and the storage class is created.

```
# kubectl get pods | grep nfs-client
vigilant-grizzly-nfs-client-provisioner-59887f6dd-86m7p   1/1     Running   0          2d14h

# kubectl get sc | grep nfs-client
nfs-client                 cluster.local/vigilant-grizzly-nfs-client-provisioner   2d14h
```



##4. Testing

Let's request a Persistent Volume Claim (PVC). Create and apply the following K8s object,

```
kubectl create -f - <<ZZZ
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-client
  resources:
    requests:
      storage: 2Gi
ZZZ
```

Watch the PVC is bound,

```
# kubectl get pvc | grep pvc-nfs-claim
pvc-nfs-claim   Bound    pvc-ed1a01ee-51a0-11e9-9d79-06bbfc5cbe29   2Gi        RWO            nfs-client     28s
```

Now let's create a test deployment using that PVC.

```
kubectl create -f - <<ZZZ
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-testing
  labels:
    app: nfs-testing
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-testing
  template:
    metadata:
      labels:
        app: nfs-testing
    spec:
      containers:
      - name: nfs-testing
        image: busybox
        imagePullPolicy: IfNotPresent
        command: ["sh"]
        args: ["-c", "while true; do sleep 60; done"]
        volumeMounts:
          - name: pv-data
            mountPath: /data
      volumes:
        - name: pv-data
          persistentVolumeClaim:
            claimName: pvc-nfs-claim
ZZZ
```

In the deployment, we define a volume using the PVC created in the previous step, then mount it into the container to the path of /data.

Wait for the pod is running, then

```
# kubectl get pods | grep nfs-test
nfs-testing-5f9889c98-v5dx6                               1/1     Running   0          38s
```

Exec into it, check out the /data file system

```
# kubectl exec -it nfs-testing-5f9889c98-v5dx6 -- sh
Defaulting container name to nfs-testing.
Use 'kubectl describe pod/nfs-testing-5f9889c98-v5dx6 -n default' to see all of the containers in this pod.
/ # cd /data
/data # df -h .
Filesystem                Size      Used Available Use% Mounted on
169.50.201.243:/data/default-pvc-nfs-claim-pvc-ed1a01ee-51a0-11e9-9d79-06bbfc5cbe29
                         98.1G     14.2G     78.8G  15% /data
```

Create a file and validate the content,

```
# echo $(hostname) > abc
/data # cat abc
nfs-testing-5f9889c98-v5dx6
```

**Exit**, and delete the current pod, a new pod should be re-created. Exec into it again, the file should be persisted with the last content.

```
# kubectl delete pods nfs-testing-5f9889c98-v5dx6
pod "nfs-testing-5f9889c98-v5dx6" deleted

# kubectl get pods | grep nfs-test
nfs-testing-5f9889c98-6g9tq                               1/1     Running   0          55s

# kubectl exec -it nfs-testing-5f9889c98-6g9tq -- sh
Defaulting container name to nfs-testing.
Use 'kubectl describe pod/nfs-testing-5f9889c98-6g9tq -n default' to see all of the containers in this pod.
/ # 

/ # cd /data
/data # cat abc
nfs-testing-5f9889c98-v5dx6
/data # 

```

This is the end.

![icp000](images/icp000.png)







